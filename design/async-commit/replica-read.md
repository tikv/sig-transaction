# Compatibility between async commit and replica read

<sub>This document was originally written for TiFlash developers</sub>

## What is async commit? 

It’s an optimization to the original 2PC.

Success of a transaction is returned to the user as soon as the first phase (aka the prewrite phase in Percolator) finishes successfully. The second phase (the commit phase) is done asynchronously. So the latency is reduced.

## What is the most important change?

The commit timestamp may be calculated by TiKV. Every read should update the `max read TS` on TiKV with its snapshot TS. On prewrite, `min commit TS` is set to at least `max read TS + 1`. Then, we can make sure the commit TS of the transaction will be larger than the snapshot TS of any previous reader. In other words, we can guarantee snapshot isolation.

## What is the problem for “replica read”?

Replica read does not update the `max read TS`.

There is a time gap between setting the “min commit TS” in the lock and the lock being applied to the raft store. These unapplied locks are saved in memory temporarily. So readers must see these in-memory locks which only exist on the leader.

## What is the solution?

Protocol change: https://github.com/pingcap/kvproto/pull/665

Two extra fields are added to the ReadIndex RPC request:

- `start_ts` is the snapshot TS of the replica read. It updates the `max read TS` on the TiKV of the leader.
- `ranges` are the key ranges to read. TiKV will check if there are memory locks in the given ranges which should block the read. If any of such locks are found, it is returned as the `locked` field in the response.
    TiFlash already uses the ReadIndex RPC for replica read. TiKV can use the similar way for replica read because it is not so easy to support it through the raft layer from the engineering perspective.

More design documents about async commit: https://github.com/tikv/sig-transaction/tree/master/design/async-commit

