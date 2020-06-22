# Parallel Commit (initial design)

This document is focussed on the initial increments of work to implement a correct, but non-performant version of parallel commit.

## Goals

* Reduce latency of transactions by moving a round trip from client to server from before reporting success to the user to after reporting success to the user.
* Not to significantly reduce throughput or increase latency of reads or prewrite.

## Requirements and constraints

* Preserve existing SI, linearizability, and RC properties of transactions.
* TiDB can report success to the user after all prewrites succeed; before sending further messages to TiKV.
* Backwards compatible (to be specified).

## Known issues and solutions

Please see [this doc](parallel-commit-known-issues-and-solutions.md).

## Implementation

### TiDB

* The user should opt-in to parallel commit; we assume the user has opted in for all nodes for the rest of this document.
* Each prewrite response will include a `min_commit_ts`, TiDB will select the largest `min_commit_ts` as the final `commit_ts` for the transaction.
* TiDB can return success to the client before sending the commit message but after receiving success responses for all prewrite messages.
* If an operation fails because it is locked, TiDB must query the primary lock to find a list of secondaries (a `CheckTxnStatus` request). It can then send a batch get request to each region to get all secondary locks and the `min_commit_ts` in those locks. If all locks are got, it can send the commit message as above. If any lock is not present or rolled back, then the transaction should be rolled back.

### TiKV

* The information stored with each primary key lock should include all keys locked by the transaction and their status.
* When a prewrite is received, lock the keys with a preliminary ts of the start ts. Query PD for a timestamp, this is returned to TiDB as the `min_commit_ts` and stored in the lock data for the primary key.
* When a read is woken up, ensure we check the commit ts, since we might want the old value (I think we do this anyway).



### Protobuf changes

Prewrite requests need to notify TiKV that a transaction is using parallel commit

```diff
  message PrewriteRequest {
      // ...
+     bool use_parallel_commit = ...;
+     repeated bytes secondaries = ...;
  }
```

And the response carries some information that helps the client to continue finalizing the transaction:

```diff
  message PrewriteResponse {
      // ...
+     // 0 if the min_commit_ts is not ready or any other reason that parallel
+     // commit cannot proceed. The client can then fallback to normal way to
+     // continue committing the transaction if prewrite are all finished.
+     uint64 min_commit_ts = ...; 
  }
```

We also need to be able to clean up dead parallel commit transactions:

```diff
  message CheckTxnStatusResponse {
      // ...
+     bool use_parallel_commit = ...;
+     repeated bytes secondaries = ...;
  }
```

## Evolution to long term solution

The framework of parallel commit will be in place at the end of the first iteration. In a later iteration we should improve the temporary locking mechanism, See [parallel-commit-solution-ideas.md](parallel-commit-known-issues-and-solutions.md) for possible improvements.

### Open questions

* There is a problem if the commit_ts calculated during a resolve lock is > pd’s latest ts + 1, that is separate from the problem of non-unique timestamps (but has the same root cause). ([#21](https://github.com/tikv/sig-transaction/issues/21))
* Replica read. ([#20](https://github.com/tikv/sig-transaction/issues/20))
* Interaction with CDC. ([#19](https://github.com/tikv/sig-transaction/issues/19))
* Interaction with binlog. ([#19](https://github.com/tikv/sig-transaction/issues/19))
* Missing schema check.
* Interaction with large transactions (there should be no problem, we must ensure that other transactions don't push a parallel commit transaction's commit_ts).
* Heartbeats for keeping parallel commit transactions alive.

### Refinements

#### TiDB

* TiDB should choose whether or not to use parallel commit based on the size of the transaction

#### TiKV

* When using a preliminary lock, try to wake up reader when we have a commit ts.
* Use a memlock rather than writing a preliminary lock

Refinement 1

* For each node, store a `local_ts` (aka `max_ts`), this is the largest ts seen by the node or issued by the node + 1. It could be kept per-region if there is lots of contention, but since it a fairly simple lock-free (but not wait-free) update, I would not expect it to be too bad.
  - Note that this is basically a Lamport Clock version of the timestamp, i.e., it is an approximation to PD's current TS.
* TiDB fetches a timestamp from PD for all prewrites (`min_commit_ts`). TiKV compares `min_commit_ts` to `local_ts`, if `local_ts` is greater than `min_commit_ts`, it must fetch a new timestamp from PD, otherwise it can reuse `min_commit_ts`.

Refinement 2

* Use the `local_ts` as the `min_commit_ts`.
* Note that this scheme causes duplicate time stamps, and requires one of the proposed solutions.

Refinement 3

* The user can opt-in to non-linearizability. In that case we use the extended timestamp format described in [parallel-commit-known-issues-and-solutions.md](globally-non-unique-timestamps.md#solution-6-extend-the-timestamp-format).
* If the user does not opt-in, then use the refinement 1 scheme.

#### 1PC

If there is just a single prewrite then TiDB can set a flag on the request, then TiKV can just use the `min_commit_ts` as the `commit_ts` and commit the transaction without TiDB sending the final commit message (or taking the risk of )

#### `max_read_ts` approach

The coarsest granularity is to maintain `max_read_ts` and `min_commit_ts` per region.

The per-range approach:

* For each region, store in memory (note this general mechanism should be abstracted using a trait so it can be easily upgraded to per-key or other form of locking):
  - A structure of `min_commit_ts`s, a map from each in-progress transaction to the minimum ts at which it may be committed.
  - `max_read_ts`: the largest `start_ts` for any transactional read operation for the region (i.e., this value is potentially set on every read).
  - When a TiKV node is started up or becomes leader, `max_read_ts` is initialised from PD with a new timestamp.
  - When a prewrite is processed, TiKV records the current `max_read_ts + 1` as the `min_commit_ts` for that transaction. `min_commit_ts` is recorded in each key's lock data structure.
  - When a prewrite is finished, its entry is removed from the `min_commit_ts` structure. If the prewrite is successful, the `min_commit_ts` is returned to TiDB in the response.
  - When a read is processed, first it sets the `max_read_ts`, then it checks its `start_ts` against the smallest `min_commit_ts` of any current transaction in the read range. It will block until its `start_ts >= min(min_commit_ts)`
* Use `Mutex<Timestamp>` for `max_read_ts`

Further refinement:

* Per-key, rather than per-region, `min_commit_ts` and `max_read_ts`
* Lock-free `max_read_ts` rather than using a mutex.

##### Handling non-unique timestamps

See [parallel-commit-known-issues-and-solutions.md](globally-non-unique-timestamps.md) for discussion.

There is a possibility of two transactions having the same commit_ts, or of one transaction’s start_ts to be equal to the other’s commit_ts. We believe conflicts in the write CF between two commits are not possible. However, if one transaction's `start_ts` is another's `commit_ts` then rolling back the first transaction would collide with committing the second. We believe this isn't too serious an issue, but we will need to find a backwards compatible change to the write CF format. We do not know if there are problems due to non-unique timestamps besides the conflict in write CF.


## Testing

## Staffing

The following people are available for work on this project (as of 2020-06-15):

* Zhenjing (@MyonKeminta): minimal time until CDC project is complete
* Zhaolei (@youjiali1995): review + minimal time
* Nick (@nrc): approximately full time
* Yilin (@sticnarf): approximately 1/2 time
* Fangsong (@longfangsong): approximately full time after apx one month (although work to be finalised)
* Rui Xu (@cfzjywxk): apx 1/3 initially

RACI roles:

* Responsible:
  - Nick
  - Rui Xu
  - Yilin
* Accountable: Shuaipeng
* Consulted (expect this list to shrink):
  - Zhenjing
  - Zhaolei
  - Rui Xu
  - Liu Wei
  - Liqi
  - Evan Zhou
* Informed (expect this list to grow):
  - #sig-transaction
  - this month in TiKV
