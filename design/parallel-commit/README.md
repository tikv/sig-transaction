# Parallel commit

This directory contains design documentation for parallel commit. Design is work-in-progress and implementation has not yet started.

## Overview

The key idea is that we can return success to the user (from TiDB) when all prewrites have succeeded, because at that point we know that commit will not fail. By returning at this point we save a round trip between TiDB and TiKV which includes a consensus write of the commit.

This modification is sound because the source of truth for whether a transaction is committed is considered to be distributed among all locks.


## Protocol

### Phase 1: reads and writes

The client (TiDB) sends `lock` messages to the server (TiKV) for each `SELECT ... FOR UPDATE`. Each lock message has a `for_update_ts`.

Modifications are collected in a buffer and sent as a `prewrite` message. No reads are permitted after prewrite.

The client gets a start TS for the whole transaction which is included with every message.

For both messages, the server checks validity and writes locally. It then sends an `ack` back to the client. For a read, the read value is returned with the ack. Then the server writes the lock and/or modifications to the RaftStore (a 'consensus write'). After the consensus write completes, the server sends a `response` to the client.

For every lock and modification, the server writes a lock to the key's lock CF; each lock stores a reference to the transaction's primary key. For modifications, we also store a value in the default CF.

For the primary key's lock, we store a list of keys in the transaction and their status (whether the key is locked locally, across all nodes, or unlocked), plus an overall status for the transaction.

TODO multiple regions.

### Phase 2: finalisation

When the client has `response`s (not `ack`s) for every message in a transaction, it sends a single `finalise` message to the server. The client considers the transaction complete when it sends the `finalise` message, it does not need to wait for a response. The client obtains the commit ts from PD for the finalise message.

When the server receives a `finalise` message, it *commits* the transaction. Committing is guaranteed to succeed.

Possible optimisation: the server could finalise the transaction when the prewrite completes without involving the client (see the resolve lock section).

Writes are added to each key which is written, with the commit ts from the primary key. The server unlocks all keys, primary key last. 

### Reads

On read, if a key is locked, then we must look up the primary key and it's lock which holds the transaction record. We wait for the lock's ttl to expire (based on the read's `for_update_ts`). After the ttl expires, if the key is unlocked, the read can progress. Otherwise we must resolve the lock in some way.

### Resolve lock

(Called TSRP in CRDB).

Resolve lock determines a transaction's commit status. If the transaction has been rolled back or committed, there is nothing to do. Otherwise, if every consensus write has succeeded, the transaction is committed. Otherwise, the transaction is considered to have timed out and is rolled back.

First we check the txn's state as recorded in the primary key. If it is written to Raft, then we can finalise (something must have failed in finalisation or finalisation was never received). (I think this is only an optimisation, we could skip this step and go straight to the next).

Otherwise, we check each lock, if all locks have state consensus write, then we run finalisation. If any lock is only written locally or has failed, then we must rollback the transaction. NOTE: this might require communicating with other nodes due to keys in other regions.

### Rollback

For each modification in a transaction, add a rollback write to the key and remove the lock. For each read, remove the lock. The primary lock should be removed last.

TODO partial rollback with for_update_ts


## Issues

See also [parallel-commit-solution-ideas.md](parallel-commit-solution-ideas.md).

### Commit timestamp

In the happy path, there is no problem. However, if the finalise message is lost then when we resolve the lock and the transaction needs committing, then we need to provide a commit timestamp. Unfortunately it seems there is no good answer for what ts to use.

Consider the following example: transaction 1 with start_ts = 1, prewrite @ ts=2, finalise @ ts=4. Transaction 2 makes a non-locking read at ts = 3. If all goes well, this is fine: the transaction is committed by receiving the finalise message with commit_ts = 4. If the read arrives before commit, it finds the key locked and blocks. After the commit it will read the earlier value since its ts is < the commit ts.

However, if the finalise message is lost, then we must initiate a 'resolve lock' once the lock times out. There are some options; bad ideas:

* Use the prewrite ts: this is unsound because the ts is less than the read's ts and so the read will see different values for the key depending on whether it arrives before or after the commit happening, even though its ts is > than the commit ts. That violates the Read Committed property.
* Transaction 2 returns an error to TiDB and TiDB gets a new ts from PD and uses that as the commit_ts for the resolve lock request. This has the disadvantage that non-locking reads can block and then fail, and require the reader to resolve locks. This timestamp is also later than the timestamp that transaction 1's client thinks it is, which can lead to RC violation.
* Get a new ts from PD. This has the problem that TiDB may have reported the transaction as committed at an earlier times stamp to the user, which can lead to RC violation.

Possibly good ideas:

* Record the 'max read ts' for each key. E.g., when the read arrives, we record 3 as the max_read_ts (as long as there is no hight read timestamp). We can then use `max_read_ts + 1` as the commit ts. However, that means that timestamps are no longer unique. It's unclear how much of a problem that is. If it is implemented on disk, then it would increase latency of reads intolerably. If it is implemented in memory it could use a lot of memory and we'd need to handle recovery somehow (we could save memory by storing only a per-node or per-region max read ts).
* Use a hybrid logical clock (HLC) for timestamps. In this way we can enforce causal consistency rather than linearisability. In effect, the ordering of timestamps becomes partial and if the finalise message is lost then we cannot compare transaction 1's timestamps with transaction 2's timestamps without further resolution. Since this would require changing timestamps everywhere, it would be *a lot* of work. Its also not clear exactly how this would be implemented and how this would affect transaction 2. Seems like at the least, non-locking reads would block.


In order to guarantee SI, commit ts need to satisfy:
* It is larger than the start ts of all transactions that have not read the lock.
* It can be larger or smaller than the transaction start ts that are executed at the same time and read the lock.
* It is smaller than the start ts of subsequent transactions.

Consider all cases of Parallel Commit:
* Normal execution process: After Prewrite is completed, it will be returned to the client response. If it is to take the commit ts asynchronously afterwards, it is possible that after the client receives the success, it initiates another transaction to obtain the start ts, and then the commit ts is obtained asynchronously. Not satisfied 3 anymore. The solutions are as follows: after obtaining commit ts and returning success, then commit asynchronously.
* After Prewrite succeeds, tidb hangs, and the transaction is successfully committed. How to determine commit ts? Of course, you can't get a new one directly from pd, it will not satisfy 3, and also negate the above solution 1, you need to have persistent information to determine the commit ts.

Parallel Commit is considered to be a successful commit after Prewrite is successful. All prewrites form a barrier, and transactions after this must be able to see it, and transactions before this can not see it. How to guarantee this? You need tikv to maintain max start ts. At the same time, prewrite writes to the lock and returns to tidb. Use max(max start ts…) + 1 to submit. When resolve lock, you can recalculate commit ts.

There is a very serious problem, prewrite takes time, resulting in the max start ts written by prewrite can not meet the requirements (violation of 1), only the max start ts after the write is successful.

The essence of solving this type of problem is to postpone the operation that may cause errors until there are no errors. The solutions are:
* The region records min commit ts, which is the smallest commit ts of the parallel commit transaction currently in progress. If the start ts of the read request is greater than min commit ts, it blocks until min commit ts is greater than start ts. That is, the read request that may have an error is blocked to ensure that there is no error before executing.
* Method 1 The granularity is too large, which is the region level, and the range can be divided within the region. The finest granularity is the key level, which is the following method:
  - The lock is written to memory first, then max start ts is obtained and then written to raftstore. When reading, first read the lock in the memory. After successfully writing to raftstore, the lock in mem is cleared, and the lock corresponding to the region is cleared when the leader switches.
  - Use rocksdb as the storage medium for memory lock, first write to rocksdb, then write to raftstore. The implementation is also simple, and the effect is the same as a.

### Initiating resolve lock

As touched upon in the commit timestamp section above, it is not clear how resolve lock should be initiated. There are two options:

* TiKV resolves locks automatically.
* TiKV returns to TiDB which instantiates resolve lock.

The TiKV approach is faster, but it means we have to get a timestamp from PD and that a read might block for a long time. The TiDB approach takes a lot longer, but is more correct.

### Replica read

The above Commit Ts calculation does not consider the replica read situation, consider the following scenario:
The leader's maxStartTs is 1, and parallel commit selects 1 + 1 = 2 as commitTs.
The startTs of replica read is 3, and it should either see the lock or the data with commitTs of 2. However, due to log replication, replica read may fail to read the lock, which will destroy snapshot isolation.

The solution is the same as prewrite solution 1:
The read index request carries start ts. When region’s min commit ts < req’s start ts, it is necessary to wait for the min commit ts to exceed start ts before responding to the read index.


### Change of leader

The above structure is stored in the leader memory. Although there is no such information on the replica, it will interact with the leader, so it is easy to solve. How to solve the transfer leader? No need to solve, because the new leader must submit an entry for the current term to provide read and write services, so all the information in the previous memory will be submitted, and this part of the information is no longer needed.

It should be noted that if the above scheme is adopted, this part of the memory information and pending requests must be processed when the leader changes.


### Blocking reads

TODO

### Schema check

The existing 2pc process checks the schema version before issue the final commit command, if we do parallel commit, we don't have a chance to check the schema version. If it has changed, we may break the index/row consistency.

The above issue only happens if the transaction prewrite phase is too long, exceeds the 2 * ddl_lease time.

## 1PC

TODO

## Possible optimisations

There are restrictions on the size of the transaction. For example, if the key involved in the transaction is less than 64, parallel commit is used, or a hierarchical structure is adopted. The primary lock records a few secondary locks, and these secondary locks record other secondary locks respectively. It is easy to implement, just recursion, and the cost of failure recovery needs to be considered.

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
- [nrc's modelling effort](https://github.com/nrc/parcom)
