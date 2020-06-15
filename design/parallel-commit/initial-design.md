# Parallel Commit (initial design)

This document is focussed on the initial increments of work to implement a correct, but non-performant version of parallel commit.

## Goals

* Reduce latency of transactions by moving a round trip from client to server from before reporting success to the user to after reporting success to the user.
* Not to significantly reduce throughput or increase latency of reads or prewrite.

## Requirements and constraints

* Preserve existing SI and RC properties of transactions.
* TiDB can report success to the user after all prewrites succeed; before sending further messages to TiKV.
* Backwards compatible (to be specified).

## Known issues and solutions

Please see [this doc](./parallel-commit-known-issues-and-solutions.md).

## Protocol design

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
+     // 0 if the max_read_ts is not ready or any other reason that parallel
+     // commit cannot proceed. The client can then fallback to normal way to
+     // continue committing the transaction if prewrite are all finished.
+     uint64 max_read_ts = ...; 
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

## First iteration

## Evolution to long term solution

## Staffing

The following people are available for work on this project (as of 2020-06-15):

* Zhenjing (@MyonKeminta): minimal time until CDC project is complete
* Zhaolei (@youjiali1995): review + minimal time
* Nick (@nrc): approximately full time
* Yilin (@sticnarf): ???
* Fangsong (@longfangsong): approximately full time after apx one month (although work to be finalised)
* Rui Xu (@cfzjywxk): ???

RACI roles:

* Responsible:
  - Nick
  - ???
* Accountable: Shuaipeng
* Consulted:
  - Zhenjing
  - Zhaolei
  - Yilin
  - Rui Xu
  - Liu Wei
  - Liqi
  - Evan Zhou
* Informed:
  - #sig-transaction
  - this month in TiKV
