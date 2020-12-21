# Async commit design spec

Includes one phase commit (1pc).

## Protocol

### Commit

The transaction is composed as usual. When prewriting, the client can set `use_async_commit` and/or `try_one_pc` to use async commit or 1pc, respectively. If using async commit, the client must specify all keys other than the primary key in the message sent to the region containing the primary key (in `secondaries`).

If async commit was used, the response will include a non-zero `min_commit_ts`. If 1pc was used, it will include a non-zero `one_pc_commit_ts`. If either was used, then success can be returned to the client if all prewrites succeed. In the case of 1pc, there are no commit messages. For async commit, they are all sent asynchronously.

`max_commit_ts` in `PrewriteRequest` and the `CommitTsTooLarge` error are used to support schema checking. See the TiDB section below.

```
message PrewriteRequest {
    ...
    bool use_async_commit = 11;
    repeated bytes secondaries = 12;
    bool try_one_pc = 13;
    uint64 max_commit_ts = 14;
}

message PrewriteResponse {
    ...
    uint64 min_commit_ts = 3;
    uint64 one_pc_commit_ts = 4;
}

message KeyError {
    ...
    CommitTsTooLarge commit_ts_too_large = 9;
}

message CommitTsTooLarge {
    uint64 commit_ts = 1; // The calculated commit TS.
}

```

### Recovery

The recovery workflow is necessary when a transaction encounters a lock from an async commit transaction. When the server encounters such a Lock, it returns a `KeyError::locked` error. The client can check `use_async_commit` to see if it must follow the async commit recovery protocol. If so, it sends a `CheckTxnStatusRequest` to the primary's store, and will get a list of the secondaries in the returned lock info. The client must send a `CheckSecondaryLocksRequest` to each region to cover all secondary keys. If all are locked or any are committed, then the transaction succeeded and the client can commit all the keys. If any have failed, then the transaction can be rolled back for all keys.

```
message CheckTransactionStatus {
    ...
    bool force_sync_commit = 7;
}

message CheckSecondaryLocksRequest {
    Context context = 1;
    repeated bytes keys = 2;
    uint64 start_version = 3;
}

message CheckSecondaryLocksResponse {
    errorpb.Error region_error = 1;
    KeyError error = 2;
    repeated LockInfo locks = 3;
    uint64 commit_ts = 4;
}

message LockInfo {
    ...
    bool use_async_commit = 8;
    uint64 min_commit_ts = 9;
    repeated bytes secondaries = 10;
}
```

## Clients

Implemented in TiDB, TiSpark.

The client has system variables for supporting async commit (`tidb_enable_async_commit`), supporting 1pc (`tidb_enable_1pc`), and forcing external consistency when using async commit (`tidb_guarantee_external_consistency`). The client has configuration options for setting the limits on the size (`tikv-client.async-commit.total-key-size-limit`) and number of keys (`tikv-client.async-commit.keys-limit`) in an async commit transaction. When async commit is enabled, a transaction will not use async commit if it exceeds either of the size limits. The rest of the spec assumes that async commit or 1pc are enabled.

The client must set `use_async_commit` and/or `try_one_pc` in the `PrewriteRequest`. If using async commit, then the `secondaries` field should contain the keys of all keys locked in the transaction except the primary key. If the user wants external consistency, a new timestamp is fetched from PD and used as the `min_commit_ts`. Otherwise, `min_commit_ts` is set to the maximum of the transaction's start and 'for update' timestamps plus `1` (if we get a timestamp from PD it should be greater than both of those).

When the client receives the `PrewriteResponse`, it should check the `one_pc_commit_ts` to see if the server used 1pc: if it is non-zero, then 1pc was used and that timestamp should be returned to the user with a success message. If it is zero, then 1pc failed and the client should continue with the 2pc protocol.

The client calculates the overall minimum commit timestamp from the maximum of the `min_commit_ts` of each response. If any `min_commit_ts` is zero, then async commit failed and the client should fallback to 2pc (see below). That overall minimum commit timestamp is returned to the user as the commit timestamp of the transaction with a success message.

With async 2pc, the primary key is committed asynchronously along with the secondary keys. This is done by sending the overall minimum commit timestamp as the commit timestamp in a commit message to each store.

If a transaction (async or sync) encounters a lock belonging to an async transaction, there is a new way to resolve the lock. The client first sends a `CheckTransactionStatusRequest` to the primary lock (this request should be retried since it is possible that a secondary lock exists before its primary lock does). Once the lock's TTL has expired, the response will indicate if the lock has been committed, rolled back, or is still locked. If it is still locked, then the client will use the list of secondary locks in the response. The client will send a `CheckSecondaryLocksRequest` to each lock, the server will return the state of each lock and (if appropriate) a commit timestamp. The client can then resolve locks; if any lock was rolled back or committed, then all locks should be rolled back or committed. If all keys are locked (or have never been locked) then the client can commit the transaction using the maximum of all lock's `min_commit_ts`.

If using async commit, the server may not be able to use this technique to complete the transaction. In this case, one or more locks will have their `use_async_commit` field set to false. The client then falls back to using plain 2pc. Then the client will set `force_sync_commit` to true when requesting the transaction status and the server will record the fall back to synchronous 2pc.

TODO schema check, max commit ts

## Server

Implemented in TiKV

A lock on disk or in memory has the following new fields: `use_async_commit`, whether the lock belongs to an async commit transaction, `secondaries`, a list of secondary locks only used for the primary key, and `rollback_ts` (see below).

The server maintains a `max_ts`, which is the largest timestamp observed by the server. Note: the concurrency manager was implemented as part of the async commit project, however, here we only describe parts of it actually used for async commit.

If `max_ts` has become out of date (due to some change to the region topology), then the store cannot handle the primary key of transactions using async commit, and must use full 2pc.

### Prewrite

The server iterates over all keys in the request. If a key is locked for async commit by the current transaction, and it is no longer an async commit transaction, then the lock can be overwritten. Each key is locked; for 1pc the lock is kept privately in memory, for 2pc the lock is written to storage. In either case the lock is also kept in memory by the concurrency manager. These in-memory locks are used to block reading while the transaction is live.

The `min_commit_ts` for the key is calculated as the maximum of the transaction's `start_ts + 1`, `for_update_ts + 1`, and `min_commit_ts`, and the `max_ts + 1` of the store. The `min_commit_ts` is written to the lock. The largest `min_commit_ts` of all keys is returned to the client.

Finally, if this is a 1pc transaction, then all writes are committed and any locks are released.

### Handling CheckSecondaryLocksRequest

The server iterates over the keys in the request. For each key, the server checks the status of the key. If we find a pessimistic lock, it is unlocked. If the key has been rolled back (either in handling this request or previously) or committed, then we return that to the client and stop checking keys. If all keys are locked by the expected transaction, then a list of the locks is sent to the client. Furthermore, for the first unlocked key, we will write a rollback record if necessary and wake up another job waiting on the lock.

Whenever we rollback a transaction (either here or when handling `CheckTransactionStatus`), we must take care with the situation where we rollback a key which has already been locked by another transaction (for example, if the original prewrite message were lost). If that transaction is committed, then it might 'over-write' our rollback record. To handle this, we add our start ts to the lock's `rollback_ts` list, if the lock is committed all entries in `rollback_ts` are written as rollback records to the write CF.

### Handling `CheckTransactionStatusRequest` and `ResolveLockRequest`

The protocol for handling locked keys is different for async commit transactions and regular transactions.

For regular transactions, the client will send a `CheckTransactionStatusRequest`to query the primary lock and (if the TTL has expired) rollback the lock or push forward its `min_commit_ts` . Then `ResolveLockRequest` is sent to unlock secondary keys.

For async commit transactions, the client will use `CheckTransactionStatusRequest` to query the primary lock, then `CheckSecondaryLocksRequest` to check the secondary locks, and then `ResolveLockRequest` to either commit or rollback both primary and secondary locks.

Therefore, when handling `CheckTransactionStatusRequest`s, the server must take care not to change the locks belonging to an async commit transaction.


## CDC

TODO

TODO CDC endpoint in tikv - global_min_lock_ts

## TiFlash

TODO
