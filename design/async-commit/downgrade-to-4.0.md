# Downgrade to 4.0

Async commit has made some changes to the format of the persistent data. The new format cannot be recognized by TiKV 4.0. And async commit locks cannot be processed by TiKV 4.0 correctly. Therefore, TiKV 5.0 users cannot downgrade seamlessly to 4.0 if async commit is used. Some extra procedures are needed before the downgrading.

## Resolve locks

Async commit locks must be handled by TiKV 5.0 clients. It is because in async commit, a transaction can be considered committed even if none of the locks are committed. However, TiKV 4.0 will always treat such a transaction uncommitted.

## Format changes

Async commit locks must be all resolved before downgrading, so we can ignore the format changes on the locks.

However, the records in the write CF also have format changes after async commit.

* Rollbacks may overlap with normal commits. `FLAG_OVERLAPPED_ROLLBACK` marks that a write record has an overlapped rollback.
* To deal with the compatibility between async commit and bypass-raft GC (compaction filter and green GC), `gc_fence` field is added to write records.

Before downgrading to 4.0, we must clear these newly added fields.

## Solution

The downgrading preparation should be done in an external tool.

1. Use the internal API to disable async commit.

2. Trigger a full lock resolving using the latest timestamp.

   After resolving locks, there should be no locks using the async commit protocol. And because async commit has already been disabled, no more async commit locks will appear.

3. Wait until all peers apply to the current index to guarantee that no async commit locks or no write records with `FLAG_OVERLAPPED_ROLLBACK` or `gc_fence` will be written to any of the TiKV stores.

4. Add a new debug service in TiKV for cleaning the fields in write records introduced by async commit. Run it on each TiKV store.

   It scans the whole write CF in the local TiKV. If it reads any write record with these new fields, it removes these fields and rewrite. The amount of data that needs rewriting should be very small.

5. After all write records are processed, downgrading to 4.0 shouldn't be blocked by async commit.

This solution does not need the cluster to be offline.

