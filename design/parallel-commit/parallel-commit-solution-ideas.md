# Parallel Commit Solution Ideas

We have many difficulties to overcome in order to implement Parallel Commit. This document includes ideas to solve them.

## One-phase Writing Locking Problem

In a Parallel-Commit transaction, once all prewrites is succeeded, we say the transaction is successfully committed. The commit_ts of a transaction `T` should be greater than all reads on `T`'s keys that happens before `T`'s prewrite operations, and less than transactions that starts after `T` committing.

We cannot asynchronously allocate a TSO as transaction `T`'s commit_ts after telling the client that the transaction has been finished, because it's possible that the client runs faster and got a earlier TSO to start its next transaction `T2`, so that in `T2`'s sight the previous transaction `T` didn't commit.

Neither can we tell the client `T` has been committed just after allocating TSO as `T`'s commit_ts. Because if server crashes it will never know what TS it has allocated, and it can not find the proper commit_ts anymore.

Actually, we believe we should persist some information that helps us in finding the commit_ts, and it should have been persisted when the transaction has been "successfully committed". Our current idea is persist `max_read_ts` into the lock that we prewrites, and the final commit_ts will be `max_over_all_keys{max_read_ts}+1`. However it's hard: we need to get the current value of `max_read_ts` before writing down the locks, however new reads may happens between getting `max_read_ts` and successfully writing down the lock.

We may have no perfect solution to this problem, but we have came up with some different ideas that may be possible to solve it.

### Idea 1: Memory Locking 

When we got the max read ts as `M` and then prewriting key `K` with max read ts `M`, we want to prevent other transactions from reading `K` with a ts larger than `M`. The most straightforward way is to make a in-memory lock: A Parallel-Commit prewrite should lock the keys it want to write, before reading the `max_read_ts`, and release the lock after finishing prewrite. A read operation should check the memory lock after updating `max_read_ts` and should be blocked (or return a KeyIsLocked error) when it meets the memory lock. Read operations don't need to actually acquire locks. It can proceed reading if it finds that the keys it wants to read are not locked.

The memory lock should support range-based queries, because a reading operation may need to scan a range. So the memory lock should be implemented by an ordered map (like BTreeMap, Trie, SkipList).

Therefore, the major difficulty of memory lock solution is that how to keep the performance.

### Idea 2: RocksDB-Based Lock (By @gengliqi)

This idea differs from [Idea1](#idea-1-memory-locking), only in that we are using RocksDB instead of an in-memory data structure to store locks. We avoid most performance impact if we can put it directly to LOCK_CF without Raft replication, however, it might introduce too many corner cases and troubles to resolve, considering that there are leader changing and region scheduling in TiKV. Therefore we may add a new CF to store these kind of locks. The performance comparing to [Idea1](#idea-1-memory-locking) is uncertain before we actually do a POC test.

### Idea 3: Transaction List (By @Little-Wallace)

For prewrite operations that need to exclude readings, add add them to a vector `V`. The `max_read_ts` should be acquired after the prewrite operation locks the latch of scheduler and adding itself to `V`.

For point-get operations, access latch to see if the key is being prewritten with a smaller ts. If so, block or return KeyIsLocked err. For range scanning operations, check each items in `V` to see if their range overlaps. If so, block or return KeyIsLocked err.

### Some other ideas

* Make something like HLC, which doesn't seem to be practical for us.
* ...

## Replica Read (By @gengliqi)

In the solutions to the locking problem, writing are performed on leaders, and it needs to know the `max_read_ts`. However if follower read read is enabled, the leader need to know the `max_read_ts` among all replicas of the Region by some way, and reading on followers should be blocked when the leader has an ongoing conflicting prewrite. Here is one possible solution to this problem.

The main idea is to adjust the lock's `max_read_ts` in Raft layer. It might be hard to avoid Raftstore coupling with transaction logic though.

First, readings on followers should send the read ts via the ReadIndex message to the leader, and the leader records it. When a prewrite of a Parallel-Commit transaction is being proposed in Raft layer, it's `max_read_ts` field should be updated if it's smaller than the `max_read_ts` that was recorded in Raft layer. This makes it possible to write down the `max_read_ts` among all replicas.

However this doesn't apply to the case that a ReadIndex arrives between a prewrite's proposal and committing. We can't edit `max_read_ts` from the raft log since it has been proposed, and we cannot immediately allow the follower to read before the log being applied. So secondly, we need a additional mechanism to prevent the follower from reading without the lock. 

In the current implementation (where Parallel Commit is not supported), ReadIndex returns the leader's commit index, and the follower can read only when its apply index >= leader's commit index. 
One way to solve this problem, is to let the leader returns `pc_index` after this `pc_index` log is committed if `pc_index` is greater than the commit index. `pc_index` indicates the index of proposed prewrite of Parallel-Commit transactions. This is easy to implement, but increases the latency of follower read, since the follower needs to wait for applying more logs before it's permitted to read. 
Another approach is to let the leader returns both `commit_index` and `pc_index` for ReadIndex, and the follower needs to construct a in-memory lock for received-but-not-yet-committed prewrites, and reads are permitted when the leader's `commit_index` is applied and, additionally, all logs before `pc_index` has been received. Then the read should be blocked if it tries to read keys that has been locked by the in-memory lock we just mentioned. Note that Raft learners also need to do this. If we choose this approach, TiFlash might be exclusive with parallel commit before we support this mechanism on it.

## Other issue

* Non-unique commit_ts issue
  * Possible solutions: [In this document](https://docs.google.com/document/d/1ofa9zYdb0-UmFu-uDHDLft2-G4s2SI2TJRErNRDH7O0/edit?usp=sharing) (in Chinese)
* Conflict to CDC
  * When calculating `max_read_ts`, the `resolved_ts` of CDC must be also considered. Also, ongoing prewrites should block `resolved_ts` from advancing. The design to achieve this may be not easy, since CDC is a side module in TiKV that references main module of TiKV, rather than TiKV referencing CDC module.
