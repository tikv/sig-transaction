# Async commit

Async commit is a change to the transaction protocols (optimistic and pessimistic). It allows returning success to the client once all prewrites succeed, without having to wait for a commit message. Safety is maintained by a having a more involved recovery procedure when another transaction encounters a lock which uses async commit. You can think of the commit status of a transaction as being distributed amongst all locks until the transaction is (asynchronously) committed.

Async commit is based on [parallel commits](https://www.cockroachlabs.com/blog/parallel-commits/)  in CRDB.

We expect async commit to give significant improvements in the latency of transactions. Since we remove one network round trip from blocking successful return to the client, the latency could (in theory) almost halve. On the other hand, recovery requires significantly more work, so that will cause regression of transactions which encounter locks. Under most workloads, transactions are mostly uncontested, so we expect async commit to cause a large net improvement to latency. Because we are still doing essentially the same work as before, we expect that throughput will not be expected.

Async commit for a single node is similar to one-phase commit. Full one-phase commit would skip the commit phase entirely since it must always succeed. This could be done later as an optimisation (it would not improve latency, but it would avoid the chance of needing recovery).

The known risks and disadvantages of async commit are:

* because it require O(n) memory for the primary lock where n is the number of keys in the transaction, it will not work well for large transactions. We will address this by having a limit on the number of keys in an async commit transaction.
* Interactions with tools such as binlog and CDC is complex, and there may be unsolvable incompatibilities.
* Async commit is more complex than our other transaction protocols, and since we must support the other protocols as well, there is a significant impact on code complexity.
* Async commit is a new, relatively untested protocol so there is higher than usual risk that there may be correctness issues (e.g., we believe async commit [affects](https://github.com/tikv/tikv/issues/8589) our linearizability guarantee).

Due to transaction latency being a department-wide priority, and because we think async commit can have a large impact on it, async commit is high priority work for the TiDB Arch team. Our goal is to deliver async commit in the 5.0 release. Since it is a large feature with the possibility of causing serious and subtle bugs, we aim to finish implementation by end of September 2020 to leave enough time for iterative testing and improvement.

## Who is working on it?

* Responsible: Nick Cameron, Zhenjing, Zhao Lei, Yilin, Xu Rui, Zhongyang Guan
* Accountable: Xu Rui
* Consulted: Evan Zhou, Arthur Mao
* Informed: Arch team, sig-txn, eng, TiKV newsletter.


## Design

For more detailed design docs, please see the [async commit directory](https://github.com/tikv/sig-transaction/tree/master/design/parallel-commit).


## Implementation

The implementation requires a new proto in kvproto: `CheckSecondaryLocks`, and some changes to other protos, mostly adding a minimum commit timestamp.

There are changes to TiKV's storage module to handle the `CheckSecondaryLocks` and the async commit protocol in prewrite, commit, and resolve locks. There are changes to TiDB's store/TiKV module to handle the changes to transaction handling. There are also changes to Unistore and tools such as CDC.

The main complication in the implementation is handling timestamps. If a transaction must be recovered, and it has not been committed, but all prewrites succeeded, then we must come up with a timestamp. For various reasons, this is difficult see [TODO]() for some details. Our solution requires tracking the timestamps of reads to a region and permitting non-uniqueness of timestamps (the generated commit timestamp might be the same as another transaction's start or commit timestamp).


## Progress

See the [tracking issue](https://github.com/tikv/sig-transaction/issues/36) for current status.

We are currently in implementation. Our first goal is an initial implementation which can demo'ed and benchmarked. This is effectively complete, although not all code has landed.

The next increment is to test, benchmark, fix bugs, and optimise. The goal is to be finished and polished for the 5.0 release of TiKV.

Much of the second increment is unknown since it depends on bugs and performance issues still to be discovered. Of the known work, my estimate is that we are 85% complete.

Known risks are:

* there are soundness/correctness problems with the async commit algorithm which cannot be fixed.
* Performance improvement is not as significant as expected.
* There are many problems discovered during testing which cannot be fixed in time to release on schedule.

TODO acceptance testing
