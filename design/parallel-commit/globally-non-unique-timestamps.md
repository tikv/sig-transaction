# Allow commit_ts to be non-globally unique.

This document is about the problem and the solution of the non-globally-unique `commit_ts` problem.

## The problem

in TiKV, rolling back a transaction need to leave a rollback record in order to guarantee consistency. But rollback records and commit records are both saved in Write CF, and the commit records are saved with `{user_key}{commit_ts}` as the internal key, while that of rollback records are `{user_key}{start_ts}`. Previously the `commit_ts`es are TSOs allocated from PD, which is guaranteed globally unique. However when we tries to use a calculated timestamp as the `commit_ts` to avoid the latency of PD's RPC, it's possible that a rollback record gets the same internal key as a commit record, but we need to keep them both.

## The solution

The solution is to keep the commit record, but with a `has_overlapped_rollback` flag in this case to indicate that there's a rollback happened here whose `start_ts` equals to the current record's `commit_ts`. But only "protected" rollbacks need to set the flag. Non-protected rollback records can be dropped without introducing potential inconsistency.

To do this, when performing a protected rollback operation, check the records in write CF to see if there's already a commit record of another transaction. If so, instead of writing the rollback record, add the `has_overlapped_rollback` flag to that commit record. For example, when rolling a transaction whose `start_ts = 10` on a key, but there's another commit record whose `start_ts = 5` and `commit_ts = 10` that updates the key, if we write the rollback record directly, then the commit record will be overwritten. So instead, keep the commit record, but set the `has_overlapped_rollback` flag.

It's also possible that when committing a transaction, there's already another transaction's rollback record that might be overwritten, which is not expected. An easy approach to solve this is to check whether there's a overlapping rollback record already here. But previously commit operations don't need to read Write CF, so introducing this check may significantly harm the performance. Our final solution to this case is that:

1. Considering a single key, if transaction `T1`'s rollback operation happens before transaction `T2`'s prewriting, `T1` can push the `max_read_ts` which can be seen by `T2`, so `T2`'s `commit_ts` can be guaranteed to be greater than `T1`'s `start_ts`. Therefore `T2` won't overwrite `T1`'s rollback record.
2. Considering a single key, if transaction `T1`'s rollback operation happens between `T2`'s prewriting and committing, the rollback will have no chance to affect `T2`'s `commit_ts`. In this case, `T1` can save its timestamp to `T2`'s lock. So when `T2` is committing, it can get the information of the rollback operation from the lock. If one of the recorded rollback timestamps in the lock equals to `T2`'s `commit_ts`, the `has_overlapped_rollback` flag will be set. Of course, if the `T1` finds that the lock's `start_ts` or `min_commit_ts` is greater than `T1`'s start_ts, or any other reason that implies that `T2`'s `commit_ts` is impossible to be the same as `T1`'s start_ts, then `T1` doesn't need to add the timestamp to the lock.

## The old discussion document

<details>
<summary>Here is an old document that discusses about different solutions to this problem.</summary>

This is a machine translation of a [Chinese document](https://docs.google.com/document/d/1ofa9zYdb0-UmFu-uDHDLft2-G4s2SI2TJRErNRDH7O0/edit#) by @MyonKeminta.

# Allow commit_ts to be non-globally unique.

## background

In our current implementation, both the start_ts and commit_ts of a transaction come from the PD and the big transactions we are currently doing and the single Region transactions we will continue to do. Both 1PC and Parallel Commit will cause commit_ts to be no longer global The only. Thus, in order to continue the work described above, many of the corner cases that we once did not need to deal with now need to have their behavior harmonized. Any case not covered by the test needs to be covered.
(None of the above optimizations will result in start_ts and commit_ts being equal for the same transaction)

## Behavior while reading and writing

### read

It's the question of whether a commit record is visible to a transaction with start_ts = T when it reads a commit record with commit_ts = T.
Here you can define T as start_ts/for_update_ts > T as commit_ts, i.e. the data for commit_ts = T is visible to the transaction for which start_ts = T.

### write

Also under the definition of T as start_ts > T as commit_ts, a The write transaction for start_ts = T encounters a commit record for commit_ts for T ( The lock can be successfully applied when (not Rollback). Because its commit_ts must be greater than start_ts, it can be compared to the commit record of the previous transaction. Coexistence. However, if you encounter a write record with the same timestamp that is not a transaction commit, but a Rollback record, then this write should fail.

In fact it would be simpler to simply disallow the locking in the above case, just as the current logic is, without change. The only advantage of the above is that it reduces some WriteConflict.

(Note: Now on a pessimistic lock, if the commit_ts of the existing commit record are the same as the current for_update_ts, then it is allowed to lock successfully)

### commit-commit conflict

It is possible two transactions have the same `commit_ts`. It's easy to imagine one transaction gets a `commit_ts` as `max(max_read_ts) + 1` and another gets that timestamp from PD. This is fine as long as the two transactions don't meet. If the two transactions try to write the same key, then there would be two competing values for any reads after that `commit_ts` (TiKV could not write both values into the write CF, but this is a technicality, the more fundamental problem is that there is no way for TiKV to judge which value is most recent). However, due to locking this cannot occur (depending on how the non-unique timestamps occur, it also might not be possible to create such a situation).


## Problems with Write CF's Rollback logs

The format of the Write CF key is {encoded_key}{commit_ts }, but it's different for Rollback-type records: a Rollback dropped transaction has no commit_ts, which has start_ts appended to the end of its key, so there could be something like this Phenomena.

* Transaction 1: start_ts = 10, commit_ts = 11
* Transaction 2: start_ts = 11, Rollback

In this case, transaction 2 may overwrite transaction 1's commit record when Write CF writes the rollback record, resulting in the loss of the data committed by transaction 1.

### Solution 1: Write Priority

At TiKV's transaction level, before writing a rollback, it is checked whether there is already another commit record equivalent to ts and if so, no more rollbacks are written; writing other commit records is allowed to overwrite the rollback record.

Disadvantages:

* Rollback requires an additional read operation, potentially causing a performance regression.
* Unable to block requests for pessimistic transactions that arrive late. This situation, once it arises on the primary key of a pessimistic transaction, may cause the pessimistic transaction to be correct. Impact. I think this can be addressed by treating a write with the same commit ts but different start ts as a rollback.

### Solution 2: Rollback CF

Separate the Rollback into a new CF. The downside is that the amount of work involved in such a change can be very high and compatibility issues need to be properly addressed. Also, this solution can help with another problem: https://docs.google.com/document/d/1suX8QQjI_eWc1PxI52vFWBBM9ajkRQxGNbAr_svypRo/edit?ts=5d91a36a#

This programme is being prepared for implementation. Related documents: https://docs.google.com/document/d/1SB4M19Xkv6zpZN4cbX6QlHJPtlB66VOJXuzL0ewx94w/edit#

### Solution 3: Rollback Flag

When a Rollback is found to collide with another Write record, the Rollback is treated as a A flag bit is added to the Write record with which the collision occurs, causing the Rollback information to collide with the transaction commit. Records coexist. As with Scenario 1, there is additional overhead and it is not very elegant to implement, but it solves a problem that has an impact on pessimistic matters. Question.

### Solution 4: Staggered ts

Modify the ts assignment logic so that start_ts is all odd commit_ts is all even. The downside is that it is too ungainly.

The current preference is for the Rollback CF solution, as this would incidentally solve a number of other problems:

* Problems with Rollback records and Lock records affecting query performance (if Lock-type Write records are also placed in the new CF).
* collapse rollback Issues that affect the validity of pessimistic matters in extreme cases.
* It's also part of the job to split write cf for latest and history.

### Solution 5: max_ts

Rather than maintaining `max_read_ts`, we maintain `max_ts` which is updated with every timestamp the TiKV node sees (i.e., every prewrite's `start_ts` and updated with every `commit_ts` when a transaction is committed).

### Solution 6: Extend the timestamp format

New timestamps are 128 bits. The first 64 are a local timestamp, the remaining 64 contain a specification version number for forward compatibility and a node identifier to identify the node that generated the timestamp. PD has the unique node id `0`. Old timestamps are considered equivalent to a new timestamp with `0` node id.

Each node maintains a local timestamp counter in the manner of a Lamport Clock. This value is sent to other nodes including PD with every message (or most messages). The ordering of the local timestamp only has the property that if event `a` observably precedes event `b`, then `ts(a) < ts(b)`. However, local timestamps are not globally unique and the inverse of the previous property is not true. Two timestamps with the same node id do provide the inverse property and all timestamps with the same node id gives a linear total order.

The entire timestamp is globally unique and gives a total ordering over timestamps. However, it is not linear in that it does not strictly match the ordering due to real time.

This solution easily solves the issue of write and rollback entries in the write CF. It also improves efficiency since to get a new timestamp, a node does not need to send a message to PD, it can use its local 'clock'.

However, it means we lose strict linearizability because the order of writes may not exactly match their real time ordering.

This solution is amenable to configuration, since if the node id is always 0, then we have the same properties as we do currently.

TODO - how does this interact with tools which require unique timestamps?

</details>