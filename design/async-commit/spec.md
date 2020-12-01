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

## Server

Implemented in TiKV

## CDC

## TiFlash
