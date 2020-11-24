# Transactions in TiKV

This doc has some notes on some of the terms and high-level concepts used when discussing transactions in TiKV. It is work in progress, and not yet in-depth.

The TiKV transaction system is based on part of the [Percolator](https://research.google/pubs/pub36726/) system (developed by Google). TiKV has the transactional parts, but not the observer parts of Percolator.

Read more:

* [TiDB docs](https://pingcap.com/docs/stable/transaction-overview/)
* [TinyKV docs](https://github.com/pingcap-incubator/tinykv/blob/course/doc/project4-Transaction.md)
* [Xuelian's blog](https://andremouche.github.io/tidb/transaction_in_tidb.html)

## Principles and foundations

* [Wikipedia article on isolation](https://en.wikipedia.org/wiki/Isolation_(database_systems))
* [Wikipedia article on snapshot isolation](https://en.wikipedia.org/wiki/Snapshot_isolation)
* [Jepsen post on consistency](https://jepsen.io/consistency)
* [Percolator paper](https://research.google/pubs/pub36726/)
* [Spanner paper](https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf)

## APIs

TiKV offers three kinds of API: raw, transactional, and [versioned](https://github.com/tikv/rfcs/blob/master/text/2020-03-11-versioned-kv.md) (which is still in development).

The raw API gives direct access to the keys and values in TiKV. It does not offer any transactional guarantees.

The transactional API encodes data using MVCC (see below). By collaborating between the client and the TiKV server, we can offer ACID transactions.

There is nothing preventing a client using both APIs, however, this is not a supported use case and if you do this you have to be very, very careful in order to not break the guarantees of the transactional API.

## Reads and writes

When discussing transactions, we usually talk about *reads* and *writes*. In this context, a 'write' could be any kind of modifying operation: creating, modifying, or deleting a value. In some places these operations are treated differently, but usually we just care about whether an operation modifies (a write) or doesn't modify (a read) the data.

## Two-phase commit

TODO

## Optimistic and pessimistic transactions

TiKV supports two transaction models. Optimistic transactions were implemented first and often when TiKV folks don't specify optimistic or pessimistic, they mean optimistic by default. In the optimistic model, reads and writes are built up locally. All writes are sent together in a prewrite. During prewrite, all keys to be written are locked. If any keys are locked by another transaction, return to client. If all prewrites succeed, the client sends a commit message.

Pessimistic transactions are the default in TiKV since 3.0.8. In the pessimistic model, there are *locking reads* (from `SELECT ... FOR UPDATE` statements), these read a value and lock the key. This means that reads can block. SQL statements which cause writes, lock the keys as they are executed. Writing to the keys is still postponed until prewrite. Prewrite and commit works

TODO pessimistic txns and Read Committed

### Interactions

between optimistic and pessimistic txns TODO

Read more:

* [blog post](https://pingcap.com/blog/pessimistic-locking-better-mysql-compatibility-fewer-rollbacks-under-high-load/)

## Multi-Version Concurrency Control (MVCC)

TiKV supports storing multiple values for a single database row key. Whenever TiKV stores a row key, it concatenates the raw bytes of the key with a 64-bit timestamp value. Thus, the row key has the logical format `key,timestamp1`, `key,timestamp2` and so on for its different versions. Helper functions exist in the [txn_types](https://github.com/tikv/tikv/blob/master/components/txn_types/src/types.rs) module to affix the timestamp to a given raw key, to extract only the timestamp from a versioned key and so on. This mechanism is what enables TiKV to support MVCC when co-operating clients make use of its transaction APIs such as `get()` or `get_for_update()`. Both optimistic and pessimistic transactions are supported.

*Optimistic transactions*: At the beginning of an optimistic transaction, a start timestamp (start_ts) is generated. When reading data, the transaction will only scan keys whose timestamp component is less than start_ts. This ensures that only data committed before the beginning of the transaction is read. At the end of the transaction, a commit timestamp (commit_ts) is generated. All writes made by the transaction will have this same commit_ts affixed to the raw key. The transaction will abort during write if any of the keys being written are found to be already present in the database with version > commit_ts.

TODO: Some important pieces of code to read regarding optimistic transactions are.. 

*Pessimistic transactions*: Similar to the above, a timestamp (called for_update_ts) is obtained at the beginning of the transaction. When reading data, the transaction will first attempt to lock each key that it reads, provided the key does not already have a version with timestamp > for_update_ts. If it does, a write-conflict error is returned. Otherwise, the key is locked by the current transaction. At the end of the transaction, a commit timestamp is obtained and used for the writes, just as with optimistic transactions.

TODO: Some important pieces of code to read regarding pessimistic transactions are..

## Consistency and isolation properties

TODO

## Timestamps

TODO what are timestamps? How are they represented, used, generated? AKA ts, version

### Some timestamps used in transactions

* `start_ts`: when the client starts to build the commit; used to identify a transaction.
* `commit_ts`: after successful prewrite, before commit phase.
* `for_update_ts`: TODO
* `min_commit_ts`: TODO 
* `current_ts`: TODO


## Regions

TODO

## Deadlock detection

TODO

## GC

TODO

## Retries

TODO

## Constraints and assumptions

TODO for each: why? implications, benefits

### Timestamps are supplied by the client

This decision benefits "user experience", performance and simplicity.

First, it gives users more control over the order of concurrent transactions.

For example, a client commits two transactions: T1 and then T2. 
If timestamps are supplied by the user, it can assure that T1 won't read any effects of T2 if T1's timestamp is smaller than T2's.
While if we let TiKV get the timestamp, the user cannot get this guarantee because the order of processing T1 and T2 is nondeterministic.

Second, it simplifies the system. Otherwise we have to let TiKV maintain states of all active transactions. 

Third, it is beneficial for performance. Large volume of transactions could overburden TiKV server. In addition, GC of inactive transactions is a problem.

TODO: further elaboration

### All timestamps are unique

TODO

It's true in previous versions of TiKV. Enabling 1PC or Async commit features could break this guarantee.

Multiple transactions may have identical commit timestamps. Start timestamps are still unique. One transaction must have distinct start_ts and commit_ts, unless it's rolled back. The commit_ts of a rollback record is the start_ts.

### From a user's perspective, reads never fail but might have to wait

Reads never fail in the read committed level. The client will always read the most recent committed version.

Read requests can return `KeyError` in the snapshot isolation level if the key is locked with `lock_ts` < `read_ts`. Then the client can try to resolve the lock and retry until it succeeds.


### The transaction layer does not know about region topology, in particular, it does not treat regions on the same node differently to other regions

A TiKV instance does not have to know the topology. 
The whole span of data is partitioned into regions. Each TiKV instance will only accept requests involving data lying in its regions. The client makes sure any request is sent to the right TiKV node that owns the data the request needs.

The design decouples transaction logic and physical data distribution. It makes shceduling more flexible and elastic. 
Imagine a redistribution of regions among a TiKV cluster that does not require any downtime or maintainance to either clients or TiKV instances.
PD as the scheduler can ask TiKV to redistribute regions, and send the latest region info to clients.

The overhead caused by such decoupling is extra network communication. Though clients must acquire regions' and TiKV stores' addresses from PD, these information be cached locally. If topology changes, client may failed some request and retry to refresh its cache. A long-live client should suffer little from it.

### If committing the primary key succeeds, then committing the secondary keys will never fail.

Even if the commit message sent to the secondary key fails, the lock of a secondary key contains information of its primary key. Any transaction that meets the lock can recognize its state by reading the primary key and help commit the secondary key.

This property allows returning success once the primary key is committed. Secondary keys could be committed asynchronously and we don't have to care about the result.

## New and planned technology

These features are not well-documented elsewhere, so should have more in-depth descriptions here.

* [Pipelined pessimistic transactions]()
* [Large transactions]()
* [Green GC]()
* [Async commit]()
* [1PC]()

## Glossary

* [Jepsen report](https://jepsen.io/analyses/tidb-2.1.7) on isolation and consistency in TiDB

TODO

* Column family (CF)
* Prewrite
* Commit
* 1PC
* two-phase commit (2PC)
* Rollback
* Resolve lock
* Write conflict
