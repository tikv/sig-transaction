# Transaction Handling Process

This article will introduce how transaction requests are handled in TiKV.

The urls in this article refers to the code which performs certain operation.

In a system which consists of TiDB and TiKV, the architecture looks like this:

![architecture](transaction-handling-newbie-perspective/architecture.svg)

Though client is not part of TiKV, it is also an important to read some code in it to understand how a request is handled. 

There're many implements of client, and their process of sending a request is similiar, we'll take [client-rust](https://github.com/TiKV/client-rust) as an example here.

Basically, TiKV's transaction system is based on Google's [Percolator](https://research.google/pubs/pub36726/), you are recommended to read some material about it before you start reading this.

### Begin

You'll need a client object to start a transaction.

The code which creates a transaction is [here](https://github.com/TiKV/client-rust/blob/fe765f191115d5ca0eb05275e45e086c2276c2ed/src/transaction/client.rs#L30), you can see the client includes a `PdRpcClient`, which is responsible for communicate with the pd component.

And then you can use [`Client::begin`](https://github.com/TiKV/client-rust/blob/fe765f191115d5ca0eb05275e45e086c2276c2ed/src/transaction/client.rs#L53) to start an transaction.

```rust, no_run
pub async fn begin(&self) -> Result<Transaction> {
	let timestamp = self.current_timestamp().await?;
	Ok(self.new_transaction(timestamp))
}
```

Firstly, we'll need to get a time stamp from pd, and then we'll create a new `Transaction` object by using current timestamp.

If you dive into `self.current_timestamp` , you'll find out that in fact it will put a request into [`PD::tso`](https://github.com/pingcap/kvproto/blob/d4aeb467de2904c19a20a12de47c25213b759da1/proto/pdpb.proto#L23) [rpc](https://github.com/TiKV/client-rust/blob/fe765f191115d5ca0eb05275e45e086c2276c2ed/src/pd/timestamp.rs#L66)'s param stream and receive a logical timestamp from the result stream.

The remote fuction `Tso` it defined is [here](https://github.com/pingcap/pd/blob/0df431e2751cbe6c3b982e0eeb7878a182e39e77/server/grpc_service.go#L74) in pd.

### Single point read

We can use `Transaction::get`  to get value for a certain key.

This part of code is [here](https://github.com/TiKV/client-rust/blob/fe765f191115d5ca0eb05275e45e086c2276c2ed/src/transaction/transaction.rs#L71):

```rust, no_run
pub async fn get(&self, key: impl Into<Key>) -> Result<Option<Value>> {
	let key = key.into();
	self.buffer.get_or_else(key, |key| {
		new_mvcc_get_request(key, self.timestamp).execute(self.rpc.clone())
	}).await
}
```

We'll try to read the local buffered key first. And if the local buffered key does not exist, a `GetResponse` will be sent to TiKV.

You may have known that TiKV divide all the data into different regions, and each replica of some certain region is on its own TiKV node, and pd will manage the meta infomation about where are the replicas for some certain key is.

The code above seems doesn't cover the steps which decide which TiKV node should we send the request to. But that's not ture. The code which do these jobs is hidden under [`execute`](https://github.com/tikv/client-rust/blob/b7ced1f44ed9ece4405eee6d2573a6ca6fa46379/src/request.rs#L33), and you'll find the the code which tries to get the TiKV node [here](https://github.com/tikv/client-rust/blob/b7ced1f44ed9ece4405eee6d2573a6ca6fa46379/src/pd/client.rs#L42) , and it is called by `retry_response_stream` [here](https://github.com/tikv/client-rust/blob/b7ced1f44ed9ece4405eee6d2573a6ca6fa46379/src/request.rs#L52):

```rust, no_code
fn store_for_key(
        self: Arc<Self>,
        key: &Key,
    ) -> BoxFuture<'static, Result<Store<Self::KvClient>>> {
        self.region_for_key(key)
            .and_then(move |region| self.map_region_to_store(region))
            .boxed()
    }
```

Firstly, it will use grpc call [`GetRegion`](https://github.com/pingcap/kvproto/blob/d4aeb467de2904c19a20a12de47c25213b759da1/proto/pdpb.proto#L41) in `region_for_key` to find out which region is the key in.

The remote fuction `GetRegion` it defined is [here](https://github.com/pingcap/pd/blob/2b56a4c5915cb4b8806629193fd943a2e860ae4f/server/grpc_service.go#L414) in pd.

And then we'll use grpc call [`GetStore`](https://github.com/pingcap/kvproto/blob/d4aeb467de2904c19a20a12de47c25213b759da1/proto/pdpb.proto#L31) in `map_region_to_store` to find out the major replica of region.

The remote fuction `GetStore` it defined is [here](https://github.com/pingcap/pd/blob/2b56a4c5915cb4b8806629193fd943a2e860ae4f/server/grpc_service.go#L171) in pd.

Finally we'll get a `KvRpcClient` instance, which represents the connection to a TiKV replica.

Then let's back to `retry_response_stream`, next function call we should pay attention to is  `store.dispatch`, it calls grpc function [`KvGet`](https://github.com/pingcap/kvproto/blob/d4aeb467de2904c19a20a12de47c25213b759da1/proto/TiKVpb.proto#L21).

And finally we reach the code in TiKV's repo. In TiKV, the requests are handled by [`Server` struct](https://github.com/TiKV/TiKV/blob/e3058403a0fc9a96870882bf184ac075223b4642/src/server/server.rs#L48) , and the `KvGet` will be handled by `future_get` [here](https://github.com/TiKV/TiKV/blob/1de029631e09a3f9989a468a9cb4b97ec4db440e/src/server/service/kv.rs#L1155).

Firstly we'll read the value for a key by using [`Storage::get`](https://github.com/TiKV/TiKV/blob/1de029631e09a3f9989a468a9cb4b97ec4db440e/src/storage/mod.rs#L216).

`get` function is a little bit long, we'll ignore `STATIC` parts for now, and we'll get:

```rust, no_run
pub fn get(&self, mut ctx: Context, key: Key,
    start_ts: TimeStamp) -> impl Future<Item = Option<Value>, Error = Error> {
    const CMD: CommandKind = CommandKind::get;
    let priority = ctx.get_priority();
    let priority_tag = get_priority_tag(priority);

    let res = self.read_pool.spawn_handle(
        async move {
            // The bypass_locks set will be checked at most once. `TsSet::vec` is more efficient
            // here.
            let bypass_locks = TsSet::vec_from_u64s(ctx.take_resolved_locks());
            let snapshot = Self::with_tls_engine(|engine| Self::snapshot(engine, &ctx)).await?;
            let snap_store = SnapshotStore::new(snapshot, start_ts,
                        ctx.get_isolation_level(),
                        !ctx.get_not_fill_cache(),
                        bypass_locks,
                        false);
            let result = snap_store.get(&key, &mut statistics)
                    // map storage::txn::Error -> storage::Error
                    .map_err(Error::from);
            result
        },
        priority,
        thread_rng().next_u64(),
    );
    res.map_err(|_| Error::from(ErrorInner::SchedTooBusy))
        .flatten()
}
```

This function will get a `snapshot`, and then construct a `SnapshotStore` by using the `snapshot`, and then call `get` on this `SnapshotStore`, and finally get the data we need.

(I don't really understand the `bypass_locks` part.)

Then we'll view the code of `SnapshotStore::get`, you'll see that in fact it consturcted a [`PointGetter`](https://github.com/tikv/tikv/blob/4ac9a68126056d1b7cf0fc9323b899253b73e577/src/storage/mvcc/reader/point_getter.rs#L133), and then call the `get` method on `PointGetter`:

```rust, no_run
pub fn get(&mut self, user_key: &Key) -> Result<Option<Value>> {
    if !self.multi {
        // Protect from calling `get()` multiple times when `multi == false`.
        if self.drained {
            return Ok(None);
        } else {
            self.drained = true;
        }
    }

    match self.isolation_level {
        IsolationLevel::Si => {
            // Check for locks that signal concurrent writes in Si.
            self.load_and_check_lock(user_key)?;
        }
        IsolationLevel::Rc => {}
    }

    self.load_data(user_key)
}
```

As we can see, if the required `isolation_level` is `Si`, we need to check whether there's any locks which may conflict with current get. If we find some, we'll return a  `KeyIsLocked` error:

```rust, no_run
fn load_and_check_lock(&mut self, user_key: &Key) -> Result<()> {
    self.statistics.lock.get += 1;
    let lock_value = self.snapshot.get_cf(CF_LOCK, user_key)?;

    if let Some(ref lock_value) = lock_value {
        self.statistics.lock.processed += 1;
        let lock = Lock::parse(lock_value)?;
        if self.met_newer_ts_data == NewerTsCheckState::NotMetYet {
            self.met_newer_ts_data = NewerTsCheckState::Met;
        }
        lock.check_ts_conflict(user_key, self.ts, &self.bypass_locks)
            .map_err(Into::into)
    } else {
        Ok(())
    }
}
```

And then we'll use `PointGetter`'s `load_data`  method to load the value.

Now we have the value in `GetResponse`, but the client still need to resolve the locked keys. This will still be handled in  `retry_response_stream`.

#### Resolve locks

First, we'll use `take_locks` to take the locks we met, and then we'll use `resolve_locks` to try to resolve them:

```rust, no_run
pub async fn resolve_locks(
    locks: Vec<kvrpcpb::LockInfo>,
    pd_client: Arc<impl PdClient>,
) -> Result<bool> {
    let ts = pd_client.clone().get_timestamp().await?;
    let mut has_live_locks = false;
    let expired_locks = locks.into_iter().filter(|lock| {
        let expired = ts.physical - Timestamp::from_version(lock.lock_version).physical
            >= lock.lock_ttl as i64;
        if !expired {
            has_live_locks = true;
        }
        expired
    });

    // records the commit version of each primary lock (representing the status of the transaction)
    let mut commit_versions: HashMap<u64, u64> = HashMap::new();
    let mut clean_regions: HashMap<u64, HashSet<RegionVerId>> = HashMap::new();
    for lock in expired_locks {
        let primary_key: Key = lock.primary_lock.into();
        let region_ver_id = pd_client.region_for_key(&primary_key).await?.ver_id();
        // skip if the region is cleaned
        if clean_regions
            .get(&lock.lock_version)
            .map(|regions| regions.contains(&region_ver_id))
            .unwrap_or(false)
        {
            continue;
        }

        let commit_version = match commit_versions.get(&lock.lock_version) {
            Some(&commit_version) => commit_version,
            None => {
                let commit_version = requests::new_cleanup_request(primary_key, lock.lock_version)
                    .execute(pd_client.clone())
                    .await?;
                commit_versions.insert(lock.lock_version, commit_version);
                commit_version
            }
        };

        let cleaned_region = resolve_lock_with_retry(
            lock.key.into(),
            lock.lock_version,
            commit_version,
            pd_client.clone(),
        )
        .await?;
        clean_regions
            .entry(lock.lock_version)
            .or_insert_with(HashSet::new)
            .insert(cleaned_region);
    }
    Ok(!has_live_locks)
}
```

First we find all the locks which are expired, and resolve them one by one. 

Then we'll get `lock_version`'s corresponding `commit_version` (might be bufferd), and use it to send `cleanup_request`.

It seems that `Cleanup` is deprecated after 4.0 , then we'll simply igonre it.

And then it is the key point: [`resolve_lock_with_retry`](https://github.com/tikv/client-rust/blob/9d1256fba6b2eb0bf8b07f2c92573011e985925c/src/transaction/lock.rs#L75), this function will construct a  `ResolveLockRequest`, and send it to TiKV to execute.

Let's turn to TiKV's source code, you'll find out that this request will be casted into a `TypedCommand`, and be executed by `sched_txn_command`.

According to whether the `key` on the request is empty, `ResolveLockRequest` will be casted into `ResolveLock` or `ResolveLockLite`. The difference between those two is that `ResolveLockLite` will only handle the locks `Request` ask for resolve, while `ResolveLock` will handle a whole region.

It is really hard to find where the `ResolveLock` command is handled. It took me a long time to find out that it has 2 parts: One is [here](https://github.com/TiKV/TiKV/blob/82d180d120e115e69512ea7f944e93e6dc5022a0/src/storage/txn/process.rs#L416), which is resposible for read out the locks, and the other is [here](https://github.com/TiKV/TiKV/blob/82d180d120e115e69512ea7f944e93e6dc5022a0/src/storage/txn/process.rs#L775), which is responsible for the release work.

These two code part uses `MvccTxn` and `MvccReader`, we'll explain them later in another article.

These [Comments](https://github.com/TiKV/TiKV/blob/82d180d120e115e69512ea7f944e93e6dc5022a0/src/storage/txn/commands.rs#L520) here has a good intruduce of what `ResolveLock` do.

And then we can go back to client-rust's `resolve_locks`, and continue with the other `expired_locks`.

And then, the result value is returned. (Finally!)

Let's summerize the process with a dataflow diagram.

![single-point-get-dfd](transaction-handling-newbie-perspective/single-point-get-dfd.svg)

### Scan

(WIP)

### Write

(WIP)

### Commit

(WIP)

### Rollback

(WIP)