# Documentation

[Open issues](https://github.com/tikv/sig-transaction/labels/documentation)

## Technologies in TiKV

### Transactions overview

* [TiDB docs](https://pingcap.com/docs/stable/transaction-overview/)
* [TinyKV docs](https://github.com/pingcap-incubator/tinykv/blob/course/doc/project4-Transaction.md)
* [Xuelian's blog](https://andremouche.github.io/tidb/transaction_in_tidb.html)

### Pessimistic locking

* [blog post](https://pingcap.com/blog/pessimistic-locking-better-mysql-compatibility-fewer-rollbacks-under-high-load/)

### Large transactions

* [blog post](https://pingcap.com/blog/large-transactions-in-tidb/)

### Other

* [Jepsen report](https://jepsen.io/analyses/tidb-2.1.7) on isolation and consistency in TiDB

## Documentation in code

* Our protocol buffer specs are documented, mostly in [kvrpcpb.proto](https://github.com/pingcap/kvproto/blob/master/proto/kvrpcpb.proto).

## Principles and foundations

* [Wikipedia article on isolation](https://en.wikipedia.org/wiki/Isolation_(database_systems))
* [Wikipedia article on snapshot isolation](https://en.wikipedia.org/wiki/Snapshot_isolation)
* [Jepsen post on consistency](https://jepsen.io/consistency)

## Meta: docs about SIG-transaction

* [SIG documents](https://github.com/tikv/community/tree/master/sig/transaction)
