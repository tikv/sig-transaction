# Parallel Commit (initial design)

This document is focussed on the initial increments of work to implement a correct, but non-performant version of parallel commit.

## Goals

* Reduce latency of transactions by moving a round trip from client to server from before reporting success to the user to after reporting success to the user.
* Not to significantly reduce throughput or increase latency of reads or prewrite.

## Requirements and constraints

* Preserve existing SI and RC properties of transactions.
* TiDB can report success to the user after all prewrites succeed; before sending further messages to TiKV.
* Backwards compatible (to be specified).

## Protocol design

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
* Accountable: Shuipeng
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
