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

## Problems with Write CF's Rollback logs

The format of the Write CF key is {encoded_key}{commit_ts }, but it's different for Rollback-type records: a Rollback dropped transaction has no commit_ts, which has start_ts appended to the end of its key, so there could be something like this Phenomena.

* Transaction 1: start_ts = 10, commit_ts = 11
* Transaction 2: start_ts = 11, Rollback

In this case, transaction 2 may overwrite transaction 1's commit record when Write CF writes the rollback record, resulting in the loss of the data committed by transaction 1.

### Solution 1: Write Priority

At TiKV's transaction level, before writing a rollback, it is checked whether there is already another commit record equivalent to ts and if so, no more rollbacks are written; writing other commit records is allowed to overwrite the rollback record.

Disadvantages:

* Rollback requires an additional read operation, potentially causing a performance regression.
* Unable to block requests for pessimistic transactions that arrive late. This situation, once it arises on the primary key of a pessimistic transaction, may cause the pessimistic transaction to be correct. Impact.

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
