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

The code which creates a transaction is [here](https://github.com/tikv/client-rust/blob/07194c4c436e393358986b84daa2ad1e41b4886c/src/transaction/client.rs#L28), you can see the client includes a `PdRpcClient`, which is responsible for communicate with the pd component.

And then you can use [`Client::begin`](https://github.com/tikv/client-rust/blob/07194c4c436e393358986b84daa2ad1e41b4886c/src/transaction/client.rs#L51) to start an transaction.

```rust, no_run
pub async fn begin(&self) -> Result<Transaction> {
	let timestamp = self.current_timestamp().await?;
	Ok(self.new_transaction(timestamp))
}
```

Firstly, we'll need to get a time stamp from pd, and then we'll create a new `Transaction` object by using current timestamp.

### Single point read

We can use `Transaction::get`  to get a single value for a certain key.

This part of code is [here](https://github.com/TiKV/client-rust/blob/fe765f191115d5ca0eb05275e45e086c2276c2ed/src/transaction/transaction.rs#L71).

We'll try to read the local buffered key first. And if the local buffered key does not exist, a `GetRequest` will be sent to TiKV.

You may have known that TiKV divide all the data into different regions, and each replica of some certain region is on its own TiKV node, and pd will manage the meta infomation about where are the replicas for some certain key is.

The code above seems doesn't cover the steps which decide which TiKV node should we send the request to. But that's not the case. The code which do these jobs is hidden under [`execute`](https://github.com/tikv/client-rust/blob/07194c4c436e393358986b84daa2ad1e41b4886c/src/request.rs#L29), and you'll find the code which tries to get the TiKV node [here](https://github.com/tikv/client-rust/blob/b7ced1f44ed9ece4405eee6d2573a6ca6fa46379/src/pd/client.rs#L42) , and it is called by `retry_response_stream` [here](https://github.com/tikv/client-rust/blob/07194c4c436e393358986b84daa2ad1e41b4886c/src/request.rs#L48):

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

The remote fuction `GetRegion` it defined is [here](https://github.com/pingcap/pd/blob/6dab049720f4c4e1a91405806fc1fa6517928589/server/grpc_service.go#L416) in pd.

And then we'll use grpc call [`GetStore`](https://github.com/pingcap/kvproto/blob/d4aeb467de2904c19a20a12de47c25213b759da1/proto/pdpb.proto#L31) in `map_region_to_store` to find out the major replica of region.

The remote fuction `GetStore` it defined is [here](https://github.com/pingcap/pd/blob/2b56a4c5915cb4b8806629193fd943a2e860ae4f/server/grpc_service.go#L171) in pd.

Finally we'll get a `KvRpcClient` instance, which represents the connection to a TiKV replica.

Then let's back to `retry_response_stream`, next function call we should pay attention to is  `store.dispatch`, it calls grpc function [`KvGet`](https://github.com/pingcap/kvproto/blob/5f564ec8820e3b4002930f6f3dd1fcd710d4ecd0/proto/tikvpb.proto#L21).

And finally we reach the code in TiKV's repo. In TiKV, the requests are handled by [`Server` struct](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/server/server.rs#L49) , and the `KvGet` will be handled by `future_get` [here](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/server/service/kv.rs#L1136).

Firstly we'll read the value for a key by using [`Storage::get`](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/storage/mod.rs#L213).

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

The `bypass_locks` part is a tricky optimize related to [large transaction](https://pingcap.com/blog/large-transactions-in-tidb/), see [this pr](https://github.com/tikv/tikv/pull/5798).

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

Now we have the value in `GetResponse`. But if the key is locked, client still need to resolve the locked keys. This will still be handled in  `retry_response_stream`.

#### Resolve locks

First, the client will take the locks we met from the response, and then we'll use `resolve_locks` to try to resolve them:

We find all the locks which are expired, and resolve them one by one. 

Then we'll get `lock_version`'s corresponding `commit_version` (might be buffered), and use it to send `cleanup_request`.

It seems that using `CleanupRequest` directly is deprecated after 4.0 , then we'll simply igonre it.

And then it is the key point: [`resolve_lock_with_retry`](https://github.com/tikv/client-rust/blob/07194c4c436e393358986b84daa2ad1e41b4886c/src/transaction/lock.rs#L74), this function will construct a  `ResolveLockRequest`, and send it to TiKV to execute.

Let's turn to TiKV's source code, according to whether the `key` on the request is empty, `ResolveLockRequest` will be converted into `ResolveLockReadPhase` + `ResolveLock` or `ResolveLockLite`. The difference between those two is that `ResolveLockLite` will only handle the locks `Request` ask for resolve, while `ResolveLock` will resolve locks in a whole region.

The handling of `ResolveLock` has 2 parts: the read phase is [here](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/storage/txn/process.rs#L122), which is resposible for read out the locks and construct the write phase command, and the write phase is [here](https://github.com/TiKV/TiKV/blob/82d180d120e115e69512ea7f944e93e6dc5022a0/src/storage/txn/process.rs#L775), which is responsible for the release work.

These two code part uses `MvccTxn` and `MvccReader`, we'll explain them later in another article.

[Comments](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/storage/txn/commands/resolve_lock.rs#L17) here gives a good intruduction of what `ResolveLock` do.

After all `expired_locks` are resolved, a new `GetRequest` is sent, and the get process will be done again until it success.

And then, the result value is returned. (Finally!)

Let's summerize the process with a dataflow diagram.

![single-point-get-dfd](transaction-handling-newbie-perspective/single-point-get-dfd.svg)

### Scan

On the client side, scan is almost the same as single point get, except that it sends a [`KvScan`](https://github.com/pingcap/kvproto/blob/5f564ec8820e3b4002930f6f3dd1fcd710d4ecd0/proto/tikvpb.proto#L22) grpc call instead of `KvGet`.

And on the TiKV side, things are a little different, firstly, the request will be handled by [`future_scan`](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/server/service/kv.rs#L1161), and then [`Storage::scan`](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/storage/mod.rs#L443)，and finally we'll find out the function which really do the job is a [`Scanner`](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/storage/mvcc/reader/scanner/mod.rs#L171), and we'll cover this part in another document. 

### Write

In fact, write just write to local buffer. All data modifications will be sent to TiKV on commit.

### Commit

Now comes the most interesting part: commit, just like what I mentioned, commit in TiKV is based on [Percolator](https://research.google/pubs/pub36726/), but there are several things that are different:

- [Percolator](https://research.google/pubs/pub36726/) depends on BigTable's single row transaction, so we must implement something alike by ourselves in TiKV.

- We need to support pessimistic transaction

  Pessimistic transaction enable TiDB to have a better MySQL compatibility, and save rollbacks under high load.

  But it introduces some other problems such as:

  - dead lock

    In optimistic transaction handling, dead lock won't happen because in the prewrite stage, a transaction would about if another transaction holds a lock it needs, and in the read stage, the locks from a dead transaction are resolved.

So let's see how TiKV deal with these things.

#### Client

From the client side, the commit process is easy, you can see we use a [`TwoPhaseCommitter`](https://github.com/tikv/client-rust/blob/fe765f191115d5ca0eb05275e45e086c2276c2ed/src/transaction/transaction.rs#L249) to do the commit job, and what it does is just as the [Percolator](https://research.google/pubs/pub36726/) paper says: [`prewrite`](https://github.com/tikv/client-rust/blob/fe765f191115d5ca0eb05275e45e086c2276c2ed/src/transaction/transaction.rs#L278), [`commit_primary`](https://github.com/tikv/client-rust/blob/fe765f191115d5ca0eb05275e45e086c2276c2ed/src/transaction/transaction.rs#L293) and finally [`commit_secondary`](https://github.com/tikv/client-rust/blob/fe765f191115d5ca0eb05275e45e086c2276c2ed/src/transaction/transaction.rs#L310).

#### AcquirePessimisticLock

This is used in the pessimistic transaction handling. It locks certain keys to prevent them from being changed by other transactions.

This one does not exists in client-rust for now, so you have to read TiDB's code [here](https://github.com/pingcap/tidb/blob/3748eb920300bd4bc0917ce852a14d90e8e0fafa/store/tikv/pessimistic.go#L58).

Basically, it sends a `PessimisticLockRequest` to TiKV, and TiKV will handle it [here](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/storage/txn/process.rs#L397), it just run `MvccTxn::acquire_pessimistic_lock` for each key to lock, which just put a lock on the key, the lock is just like the lock used in prewrite in optimistic transaction, the only differece is its type is `LockType::Pessimistic`.

And the it returns whether the lock is successful. If not, it will also [return the lock to wait for](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/storage/txn/process.rs#L447).

#### Prewrite

On TiKV side, the prewrite process happens [here in `process_write_impl`](https://github.com/tikv/tikv/blob/4a75902f266fbbc064f0c19a2a681cfe66511bc3/src/storage/txn/process.rs#L557).

The first few lines of code (`if rows > FORWARD_MIN_MUTATIONS_NUM` part) is not covered by the [`TiKV Source Code Reading blogs`](https://pingcap.com/blog-cn/tikv-source-code-reading-12/). I guess it means:

```
if there's no "write" record in [mutations.minKey, mutation.maxKey] {
	skip_constraint_check = true;
  scan_mode = Some(ScanMode::Forward)
}
```

As far as I understand, it just provides a optimized way of checking the "write" column, see [tikv#5846](https://github.com/tikv/tikv/pull/5846) for details.

And no matter whether this branch is taken, we'll construct a `MvccTxn` , and then use it to do the prewrite job for each mutation the client sent to the TiKV server.

The [`MvccTxn::prewrite`](https://github.com/tikv/tikv/blob/4a75902f266fbbc064f0c19a2a681cfe66511bc3/src/storage/mvcc/txn.rs#L563) function just do what the [Percolator](https://research.google/pubs/pub36726/) describes: check the `write` record in `[start_ts, ∞]` to find a newer write (this can be bypassed if `skip_constraint_check` is set, we can ignore this check safely in situations like import data). And then check whether the current key is locked at any timestamp. And finally use [`prewrite_key_value`](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/storage/mvcc/txn.rs#L207) to lock the key and write the value in.

##### Latches

Just as I mentioned, there's no such things like "single row transaction" in TiKV, so we need another way to prevent the key's locking state changed by another transaction during `prewrite`.

TiKV use [`Latches`](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/storage/txn/latch.rs#L125) to archieve this, you can consider it as a Map from key('s hashcode) to mutexes. You can lock a key in the `Latches` to prevent it be used by other transactions.

The latches is used in [`try_to_wake_up`](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/storage/txn/scheduler.rs#L335) , this is called before each command is executed, it will lock all the latches the commands used.

![prewrite-dfd](transaction-handling-newbie-perspective/prewrite-dfd.svg)

#### PrewritePessimistic

[`PrewritePessimistic`'s handling](https://github.com/tikv/tikv/blob/3a4a0c98f9efc2b409add8cb6ac9e8886bb5730c/src/storage/txn/process.rs#L624) is very similiar to `Prewrite`, except it:

- doesn't need to read the write record for checking conflict, because the potential conflicts have already checked during acquiring the lock
- downgrade the pessimistic lock to optimistic lock during prewrite, so the following commit process would be the same as the commit process in optmistic transaction handling
- needs to prevent deadlock

##### Dead lock handling

There won't be a dead lock in the optimistic transaction handling process, because we can know all the keys to lock during the prewrite process, so we can lock them in order.

But during the pessimistic transaction handling process, the situation is very different: when to lock a key or which keys to lock are totally decided by the user, so for example:

```
transaction A:
	lock key a;
	do some process;
	lock key b;
	do some other process;
commit
```

and

```
transaction B:
	lock key b;
	do some process;
	lock key a;
	do some other process;
commit
```

If you are unlucky, transaction A will hold the lock on `a` and try to get the lock on `b`, and transaction B will hold the lock `b` and try to get the lock on `a`, and neither of them can get the lock and continue with their jobs, so a dead lock occurred.

TiKV use deadlock detection to prevent this kind of situation.

The deadlock detector is made up with two parts: the [`LockManager`](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/server/lock_manager/mod.rs#L49) and the [`Detector`](https://github.com/tikv/tikv/blob/3a4a0c98f9efc2b409add8cb6ac9e8886bb5730c/src/server/lock_manager/deadlock.rs#L467).

Basically, these two make a *Directed acyclic graph* with the transactions and the locks it require, if adding a node may break the "acyclic" rule, then a potential deadlock is detected, a separate doc will be add to describe the [`LockManager`](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/server/lock_manager/mod.rs#L49).

#### (Do) Commit

After `prewrite` is done, the client will do the commit works: first commit the primary key, then the secondary ones, both these two kind of keys' commit are represented by the `Commit` command and handled [here](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/storage/txn/process.rs#L454). 

In the commit process we just use [`MvccTxn::commit`](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/storage/mvcc/txn.rs#L681) to commit each key, which it does is much like [Percolator](https://research.google/pubs/pub36726/) describes,.

We also collect the released locks and use it to [wake up the waiting pessimistic transactions](https://github.com/tikv/tikv/blob/17e75b6d1d1a8f1fb419f8be249bc684b3defbdb/src/storage/txn/process.rs#L513).

### Rollback

#### (Optimistic) Rollback

On the client side, [rollback](https://github.com/tikv/client-rust/blob/fe765f191115d5ca0eb05275e45e086c2276c2ed/src/transaction/transaction.rs#L327) just construct a `BatchRollbackRequest` with the keys changed in this transaction and a `start_version` which identify the transaction to be rolled back, and send it to server.

On the server side, it just call [`MvccTxn::rollback`](https://github.com/tikv/tikv/blob/1709de63b3b66f474ff757133a8a1076cf77f196/src/storage/mvcc/txn.rs#L732) [here](https://github.com/tikv/tikv/blob/1709de63b3b66f474ff757133a8a1076cf77f196/src/storage/txn/process.rs#L778), and [`MvccTxn::rollback`](https://github.com/tikv/tikv/blob/1709de63b3b66f474ff757133a8a1076cf77f196/src/storage/mvcc/txn.rs#L732) is a direct proxy to [`MvccTxn::cleanup`](https://github.com/tikv/tikv/blob/1709de63b3b66f474ff757133a8a1076cf77f196/src/storage/mvcc/txn.rs#L795).

Let's view the code in [`MvccTxn::cleanup`](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/storage/mvcc/txn.rs#L831):

The first branch in the `match` is taken when there's a lock on the key.

`!current_ts.is_zero()` is always false in the rollback situation, so we'll ignore it here.

Then we'll call [`MvccTxn::rollback_lock`](https://github.com/tikv/tikv/blob/bf716a111fde9fe8da56f8bd840c53d80c395525/src/storage/mvcc/txn.rs#L263):

First remove the value written if necessary:

```rust
if lock.short_value.is_none() && lock.lock_type == LockType::Put {
	self.delete_value(key.clone(), lock.ts);
}
```

And then put the write record.

```rust
let protected: bool = is_pessimistic_txn && key.is_encoded_from(&lock.primary);
let write = Write::new_rollback(self.start_ts, protected);
self.put_write(key.clone(), self.start_ts, write.as_ref().to_bytes());
```

And then collapse the prev rollback if necessary:

```rust
if self.collapse_rollback {
	self.collapse_prev_rollback(key.clone())?;
}
```

Finally unlock the key:

```rust
Ok(self.unlock_key(key, is_pessimistic_txn))
```

On the other hand, in [`MvccTxn::cleanup`](https://github.com/tikv/tikv/blob/1709de63b3b66f474ff757133a8a1076cf77f196/src/storage/mvcc/txn.rs#L795), when there's no lock on the key, first we'll use [`check_txn_status_missing_lock`](https://github.com/tikv/tikv/blob/1709de63b3b66f474ff757133a8a1076cf77f196/src/storage/mvcc/txn.rs#L745) to decide the status of the transaction, if the transaction has already committed, return an error, else it is ok.

#### Pessimistic Rollback

The only difference between the handling of `PessimisticRollback` and `Rollback` is `PessimisticRollback` use [`MvccTxn::pessimistic_rollback`](https://github.com/tikv/tikv/blob/1709de63b3b66f474ff757133a8a1076cf77f196/src/storage/mvcc/txn.rs#L842) here.

And the only job [`MvccTxn::pessimistic_rollback`](https://github.com/tikv/tikv/blob/1709de63b3b66f474ff757133a8a1076cf77f196/src/storage/mvcc/txn.rs#L842) is to remove the lock the transaction put on the key.

## Summary

This article gives a brief introduction on how transactions are handled in TiKV, and contain links which shows us where are the code corresponding to some certain action.

This is just a high-level and brief introduction, we did not dive very deep into several parts of the code base, eg. the mvcc part, the scheduler. But I hope this article can give you a basic view of TiKV's transaction handling system and help you to get farmiliar of some of our code base.