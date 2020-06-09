# Parallel commit

This directory contains design documentation for parallel commit. Design is work-in-progress and implementation has not yet started.

## Overview

Parallel commit extends pipelined pessimistic locking. The key idea is that we can return success to the user (from TiDB) when all prewrites have succeeded, because at that point we know that commit will not fail. By returning at this point we save a round trip between TiDB and TiKV which includes a consensus write of the commit.

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

### Commit timestamp

In the happy path, there is no problem. However, if the finalise message is lost then when we resolve the lock and the transaction needs committing, then we need to provide a commit timestamp. Unfortunately it seems there is no good answer for what ts to use.

Consider the following example: transaction 1 with start_ts = 1, prewrite @ ts=2, finalise @ ts=4. Transaction 2 makes a non-locking read at ts = 3. If all goes well, this is fine: the transaction is committed by receiving the finalise message with commit_ts = 4. If the read arrives before commit, it finds the key locked and blocks. After the commit it will read the earlier value since its ts is < the commit ts.

However, if the finalise message is lost, then we must initiate a 'resolve lock' once the lock times out. There are some options:

* Use the prewrite ts: this is unsound because the ts is less than the read's ts and so the read will see different values for the key depending on whether it arrives before or after the commit happening, even though its ts is > than the commit ts. That violates the Read Committed property.
* Transaction 2 returns an error to TiDB and TiDB gets a new ts from PD and uses that as the commit_ts for the resolve lock request. This has the disadvantage that non-locking reads can block and then fail, and require the reader to resolve locks.
* Record the 'max read ts' for each key. E.g., when the read arrives, we record 3 as the max_read_ts (as long as there is no hight read timestamp). We can then use `max_read_ts + 1` as the commit ts. However, that means that timestamps are no longer unique. It's unclear how much of a problem that is.
* Get a new ts from PD. This has the problem that TiDB may have reported the transaction as committed at an earlier times stamp to the user, which can lead to RC violation.


TODO: The problem is that, if a key is read with ts=2, and then prewritten by a transaction with start_ts=1, and there should be a way for us to guarantee the final commit_ts > 2. According to youjiali's original design, we can persist the max_read_ts when prewritting. However there's still a problem that if we are going to prewrite with max_read_ts=2, but before we finishing prewrite, anther read with ts=3 may arrive.  Here I didn't fully understand how you are going to solve this problem...


### Initiating resolve lock

### Replica read

TODO

### Blocking reads

TODO

## 1PC

TODO

## Resources

- [Zhao Lei's doc 中文 + en](https://docs.google.com/document/d/1-yn5zyn8NpqXRii9sA5wDcNHL3L0BYVaEFyD-YChX1g/edit#)
- [CockroachDB blog post](https://www.cockroachlabs.com/blog/parallel-commits/)
- [nrc's modelling effort](https://github.com/nrc/parcom)
