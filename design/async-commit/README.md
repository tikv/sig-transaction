# Async commit

This directory contains design documentation for async commit. Implementation is in progress, see [project summary](project-summary.md).

Implementation is tracked [internally](https://docs.google.com/spreadsheets/d/1W6QYTtd7iG7FWfod9c1j-apYElGiIxKRqvQeiDYpjtY/edit#) at PingCAP.


## Overview

The key idea is that we can return success to the user (from TiDB) when all prewrites have succeeded, because at that point we know that commit will not fail. By returning at this point we save a round trip between TiDB and TiKV which includes a consensus write of the commit.

This modification is sound because the source of truth for whether a transaction is committed is considered to be distributed among all locks.

The main difficulty is in choosing a timestamp for the commit ts of the transaction.

See [the design spec](spec.md) for detail.

## Protocol

### Prewrite

TiDB sends an enhanced prewrite message, with `use_async_commit` set to true, and `secondaries` set to a list of all keys in the transaction, except the primary key.

In TiKV, the secondary keys are stored in the lock data structure for the primary key.

If all prewrite messages are successful, then TiDB can send success to its client without waiting for the commit phase.

TiKV returns a `min_commit_ts` to TiDB in its prewrite response. It will be `0` if async commit could not be used. The timestamp sent to TiDB is the maximum of the timestamps for each key. The minimum commit timestamp for each key is the maximum of the max read ts (i.e., a conservative approximation of the last time the key was read), the start ts of the transaction, and the for_update_ts of the transaction.

### Commit

The commit phase is mostly unchanged, the only difference is that all commit messages are sent asynchronously, including the primary key's.

### Resolve lock

(Called TSRP in CRDB).

When resolving a lock, TiDB must find the primary key and queries it. If the key is committed, then the transaction has been committed. If the key is locked, then TiKV sends the secondary keys back to TiDB. TiDB must query all of them using a `CheckSecondaryLocks` message. If any are not committed, then the transaction is not committed and must be rolled back.


## 1PC

Async commit is a generalisation of 1pc. We can reuse some of the async commit work to implement 1pc. 1pc is only possible when all keys touched by a transaction are in the same region and binlog is not being used. TiDB then sets `try_one_pc` in the prewrite message to true.

In TiKV, a 1pc transaction can be committed in the prewrite phase, and no further action is needed. 

## Possible optimisations

There are restrictions on the size of the transaction. For example, if the key involved in the transaction is less than 64, async commit is used, or a hierarchical structure is adopted. The primary lock records a few secondary locks, and these secondary locks record other secondary locks respectively. It is easy to implement, just recursion, and the cost of failure recovery needs to be considered.

Crdb mentioned two ways to reduce the impact of recovery, and TiDB has also implemented: one is to perform commit cleanup as soon as possible when committing; the second is transaction heartbeat to prevent cleanup of alive transactions.

## Related Work

### Cockroach DB

In CRDB, parallel commit extends pipelined pessimistic locking.

crdb's transaction model is similar to TiDB in that both are inspired by percolator, but the crdb is a pessimistic transaction, every DML writes write intents, and they have many optimizations such as pipeline consensus write to reduce latency (which can also be used for pessimistic transactions). ), remain at 2PC until all write intents are written successfully on transaction commit. and update the transaction record (similar to primary key) to COMMITTED, and then returns success to the client after success.

[crdb mentions an optimization in Parallel Commits](https://www.cockroachlabs.com/blog/parallel-commits/) that avoids the 2PC. The second stage has an effect on latency, similar to that of cross-region 1PC. The idea is simple: during the transaction commit phase, update the transaction record to STAGING state and record all the keys that the transaction will modify before waiting for the write The intents and transaction record are written successfully, and can then be returned to the The client succeeds, and crdb cleans up the commit asynchronously. Since the transaction record records all the keys in the firm, it is possible to use these keys as the basis for the Information to ensure atomic submission of transactions:

* If all write intents in the STAGING state of the transaction record are written successfully, the transaction commits successfully.
* If the transaction is not in STAGING or there is no transaction record or the write intents were not written successfully, the transaction commit fails.


## Resources

- [Zhao Lei's doc 中文 + en](https://docs.google.com/document/d/1-yn5zyn8NpqXRii9sA5wDcNHL3L0BYVaEFyD-YChX1g/edit#)
- [CockroachDB blog post](https://www.cockroachlabs.com/blog/parallel-commits/)
