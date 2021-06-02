# SIG-transaction

[Rendered version](https://tikv.github.io/sig-transaction/) (Recommended)

The transactions special interest group (SIG-transaction) are a group of people interested in transactions in distributed databases. We have a focus on transactions in **TiKV and TiKV clients** (including TiDB, which use TiKV go-client), but discuss academic work and other implementations too.

## SIG activites

* talks on distributed transactions,
* a [reading group](https://tikv.org/blog/sig-txn-reading-group-nov-20/) for academic papers,
* discussion of transaction research and implementations on Slack,
* help understanding and configuring transactions in TiKV and TiDB,
* support for contributors to TiKV and related projects.


## Get involved

1. You can join us in #sig-transaction in the [TiKV community Slack](https://slack.tidb.io/invite?team=tikv-wg&channel=sig-transaction&ref=community-sig); come say hi! We use English or Chinese. (Recommended)

2. You can read or join our announcement [mailing list](https://groups.google.com/d/forum/tikv-sig-transaction), so you can stay up to date with what we're up to.

## People

See [people.md](people.md). If you want to mention us in an issue, PR, or comment, use @tikv/sig-txn.

If you have questions about the SIG or transactions in TiKV, please get in touch with either of the leaders!

## Resources

* [The TiKV blog](https://tikv.org/blog/),
* [TiDB transaction documentation](https://pingcap.com/docs/stable/transaction-overview),
* [TiDB and TiKV blog](https://dev.to/srisatya/for-readers-that-may-not-be-familiar-what-are-tidb-and-tikv-3m56).

## Repositories

* [TiKV](https://github.com/tikv/tikv),
* [TiKV Rust client](https://github.com/tikv/client-rust/),
* [TiKV Go client](https://github.com/tikv/client-go/),
* [TiKV Java client](https://github.com/tikv/client-java/),
* [kvproto](https://github.com/pingcap/kvproto) (protocol buffers which support transactions, among other things).

## Ongoing transactions work in TiKV

* [Transaction issues](https://github.com/tikv/tikv/issues?q=is%3Aopen+is%3Aissue+label%3Acomponent%2Ftransaction),
* [sig-transaction issues](https://github.com/tikv/tikv/issues?q=is%3Aopen+is%3Aissue+label%3Asig%2Ftransaction),
* [storage issues](https://github.com/tikv/tikv/issues?q=is%3Aopen+is%3Aissue+label%3Acomponent%2Fstorage) (most of these are transaction-related in some way),
* project: [Pipelined pessimistic lock](https://github.com/tikv/tikv/projects/37),
* project: [Large transactions](https://github.com/tikv/tikv/projects/36),
* project: [Green GC](https://github.com/tikv/tikv/projects/35),
* project: [Async commit](https://github.com/tikv/tikv/projects/34).
