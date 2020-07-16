# SIG-transaction

The transactions special interest group (SIG-transaction) are a group of people interested in transactions in distributed databases. We have a focus on transactions in TiKV and TiDB, but discuss academic work and other implementations too.

The SIG is in the process of starting up and currently does not have any official activity. In the near future we hope to host:

* talks on distributed transactions,
* a reading group for academic papers,
* discussion of transaction research and implementations on Slack,
* help understanding and configuring transactions in TiKV and TiDB,
* support for contributors to TiKV and related projects.


## How to get involved

You can join us in #sig-transaction in the [TiKV community Slack](https://slack.tidb.io/invite?team=tikv-wg&channel=sig-transaction&ref=community-sig); come say hi! We use English or Chinese.

You can read or join our announcement [mailing list](https://groups.google.com/d/forum/tikv-sig-transaction), so you can stay up to date with what we're up to.

If you would like to join the SIG or have any questions about it, please email sig-txn@tikv.dev.


## Repository contents

* [People](people.md) in the SIG.

Coming soon:

* Resources for learning about distributed transactions,
* notes from reading group discussions,
* slides and notes for talks and presentations,
* meeting minutes.


## People

See [people.md](people.md). If you want to mention us in an issue, PR, or comment, use @tikv/sig-txn.

### Leaders

If you have questions about the SIG or transactions in TiKV, please get in touch with either of us!

* Nick Cameron, [@nrc](https://github.com/nrc).
* Andy Lok, 骆迪安, [@andylokandy](https://github.com/andylokandy)

## Links

### Resources

* [The TiKV blog](https://tikv.org/blog/),
* [SIG documents](https://github.com/tikv/community/tree/master/sig/transaction),
* [TiDB transaction documentation](https://pingcap.com/docs/stable/transaction-overview).

### Repositories

* [TiKV](https://github.com/tikv/tikv),
* [TiKV Rust client](https://github.com/tikv/client-rust/),
* [TiKV Go client](https://github.com/tikv/client-go/),
* [TiDB](https://github.com/pingcap/tidb),
* [kvproto](https://github.com/pingcap/kvproto) (protocol buffers which support transactions, among other things).

### Ongoing transactions work in TiKV

* [Transaction issues](https://github.com/tikv/tikv/issues?q=is%3Aopen+is%3Aissue+label%3Acomponent%2Ftransaction),
* [sig-transaction issues](https://github.com/tikv/tikv/issues?q=is%3Aopen+is%3Aissue+label%3Asig%2Ftransaction),
* [storage issues](https://github.com/tikv/tikv/issues?q=is%3Aopen+is%3Aissue+label%3Acomponent%2Fstorage) (most of these are transaction-related in some way),
* project: [Pipelined pessimistic lock](https://github.com/tikv/tikv/projects/37),
* project: [Large transactions](https://github.com/tikv/tikv/projects/36),
* project: [Green GC](https://github.com/tikv/tikv/projects/35),
* project: [Parallel commit](https://github.com/tikv/tikv/projects/34).
