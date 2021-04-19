# Single Region 1PC

For transactions that affect only one region, or more strictly speaking, that can be prewritten with only one prewrite request, can be committed directly while the prewrite request is being handled, so the commit phase can be totally removed. Therefore we can get less latency and more throughput. For TiDB, indices and rows are unsally not in a same region, **so this optimization only works under a limited amount of scenarios, like sysbench oltp_update_non_index**. But in a suitable scenario, it gains significantly better performance.

We used to think about supporting 1PC before but stopped after meeting many difficulties. Now since async commit, which meets and solved mostly the same problem as 1PC, is implemented, we can continue supporting 1PC with much less effort to make.

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

We verified the improvement on a draft implementation with a brief test with following configurations, on a cluster with 3 TiKV nodes(16c 90G each) and 1 TiDB node(16c 64G):

```
sysbench oltp_update_non_index run --rand-type=uniform --db-driver=mysql --tables=8 --table-size=10000000 --time=270 --percentile=99 --report-interval=10 --threads=128
```

Here's the result:

| test                       |     qps | lat avg (ms) | lat .99 (ms) | lat max (ms)|
|----------------------------|---------|--------------|--------------|-------------|
|update_non_index<br/>1pc    | 16603.37|          7.71|         15.00|        81.50|
|update_non_index<br/>async_commit|13278.29|      9.64|         18.61|       110.50|
|update_non_index<br/>2pc    |  8892.85|         14.39|         24.83|       118.57|


## Plan

TBD
