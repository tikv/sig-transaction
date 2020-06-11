# Transactions in TiKV

This doc has some notes on some of the terms and high-level concepts used when discussing transactions in TiKV. It is work in progress, and not yet in-depth.

The TiKV transaction system is based on part of the [Percolator](https://research.google/pubs/pub36726/) system (developed by Google). TiKV has the transactional parts, but not the observer parts of Percolator.

## New and planned technology

These features are not well-documented elsewhere, so should have more in-depth descriptions here.

* [Pipelined pessimistic transactions](TODO)
* [Large transactions](TODO)
* [Green GC](TODO)
* [Parallel commit](TODO)
* [1PC](TODO)

## APIs

TiKV offers three kinds of API: raw, transactional, and [versioned](https://github.com/tikv/rfcs/blob/master/text/2020-03-11-versioned-kv.md) (which is still in development).

The raw API gives direct access to the keys and values in TiKV. It does not offer any transactional guarantees.

The transactional API encodes data using MVCC (see below). By collaborating between the client and TiKV server, we can offer ACID transactions.

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

## MVCC

TODO

## Consistency and isolation properties

TODO

## Timestamps

TODO what are timestamps? How are they represented, used, generated. AKA ts, version

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

* Timestamps are supplied by the client, TiKV does not get them directly from PD (exception: CDC).
* All timestamps are unique.
* Reads never fail (due to the transaction protocol, there might be network issues, etc. which cause failure).
  - A locking read in a pessimistic transaction may block.
  - A non-locking read will never block.
* TiKV nodes do not communicate with each other, only with a client.
* The transaction layer does not know about region topology, in particular, it does not treat regions on the same node differently to other regions.
* If committing the primary key succeeds, then committing the secondary keys will never fail.

## Glossary

TODO

* Column family (CF)
* Prewrite
* Commit
* 1PC
* two-phase commit (2PC)
* Rollback
* Resolve lock
* Write conflict
