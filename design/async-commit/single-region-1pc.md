# Single Region 1PC

In the old days we did some attempt to implement single region 1PC for single region transactions, but due to incompatibility and some technical problem it was delayed. [There is some document left at that time (note that it's very outdated)](https://docs.google.com/document/d/1Vkk8LpYbXaQ0ualdFFH35V6mv9c9RsWJu2s6nhJz9E4/edit). But now since async commit, which meets and solved the same problem as 1PC, is implemented, we can continue supporting 1PC with much less effort to make.

It's expected that by supporting single region 1PC, scenarios such as update_non_index will have better performance, lower latency and higher throughput.

* Problems already solved by supporting async commit:
  * Commit ts calculation
  * Non-unique commit ts and rollback record overlapping
  * Memory lock
  * Replica read problem
* Problems that we do not care anymore:
  * Binlog incompatibility

## Basic Design

Since async commit is implemented, it's not difficult to implement a working 1PC. However, it's still hard to make it perfectly correct and compatible with other components.

* When committing a transaction, if TiDB finds that the prewrite phase can be done with only one single request, the transaction is allowed to be committed with 1PC protocol. A field named `try_one_pc` in the prewrite request will be set to let TiKV know that 1PC is available for this transaction.
* When TiKV receives a request with `try_one_pc` set, it first handle it just like how it handles normal prewrite requests.But after generating write buffer and before writing them down to RocksDB, it will additionally check if the prewrite is fully successful, and convert the locks into commit records if so. And finally write them down to RocksDB. The `commit_ts` is `max(max_ts, start_ts, for_update_ts) + 1`. It fetches the `max_ts` while acquiring the memory lock, and the memory lock is released after applying, just like how async commit does. The final `commit_ts` will be sent back to TiDB via prewrite response.
* 1PC and async commit can be independent. When TiKV rejects to commit a transaction with 1PC, the transaction can then fallback to normal transactions, and it may become a normal 2PC transaction or an async commit transaction, according to if the async commit flag is set.

## Problems need to be solved

### Schema version checking problem

[This problem exists in async commit too](https://github.com/tikv/sig-transaction/blob/master/design/async-commit/parallel-commit-known-issues-and-solutions.md#schema-version-checking). But it's even harder to solve for 1PC, because if it's committed in TiKV, it will have no chance to check if the schema version has changed between the `start_ts` and `commit_ts`, while for async commit it can be checked after prewrite finishing.

Possible solution: When trying committing a transaction with 1PC, find a ts `one_pc_max_commit_ts` before which we can guarantee that the schema version can't change, and send it to TiKV. TiKV will reject committing if the calculated `commit_ts` exceeds the `one_pc_max_commit_ts`.

### CDC compatibility problem

CDC syncs data from TiKV by observing applying events. Prewrites and commits are distinguished, and CDC will use these events to compose complete transactions and then send them to the downstream. So when 1PC is enabled, there need to be some way for CDC to distinguish if a apply event is caused by 1PC committing, otherwise CDC will expect that a commit event must has a corresponding prewrite event to compose a complete transaction.

Possible solutions:
1. Passing a `is_1pc` flag from txn layer to apply, just like how `TxnExtra` (which is used to support CDC outputting old value) was written previously. It would be ugly.
2. When an apply batch puts write records but without deleting lockcf, we conclude that it must be a 1PC committing. This sounds like a simple way, but this may leave troubles to future. For example, if we want to introduce some new mechanisms that may amend persisted write records, we will then also need a new mechanism to distinguish the amending and 1PC committing. There's even one more problem: if the transaction being committed is a pessimistic transaction, the pessimistic lock need to be removed while writing the commit record
3. Add a field to write record that marks it's committed by 1PC. This approach makes the write record more complicated may waste some disk space.

## POC Test

WIP

## Plan

TBD
