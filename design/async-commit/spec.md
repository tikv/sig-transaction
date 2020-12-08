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

The client has system variables for supporting async commit (`tidb_enable_async_commit`), supporting 1pc (`tidb_enable_1pc`), and forcing external consistency when using async commit (`tidb_guarantee_external_consistency`). The client has configuration options for setting the limits on the size (`tikv-client.async-commit.total-key-size-limit`) and number of keys (`tikv-client.async-commit.keys-limit`) in an async commit transaction. When async commit is enabled, a transaction will not use async commit if it exceeds either of the size limits.

TODO prewrite and commit

TODO resolve

TODO fallback

TODO schema check

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
