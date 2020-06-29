# Transaction Layer Refactoring

## Motivation

At the very beginning, all transactional commands in TiKV share common procedures:

1. Acquire latches
2. Check constraints and generate modifications
3. Apply modifications to the raft store
4. Reply to RPC and release the latches

After the latches and the snapshot are acquired, all the commands depend on nothing else, so we needn't care about passing dependencies in deeply.

So the current implementation structure is natural and great: different commands are wrapped as enum variants and go through the whole process. In each step we match the type of command and do specific work.

However, as more commands and optimizations are added, this is not always the case. Some commands breaks the procedure and adds dependencies.

For example, the "pipelined pessimistic lock" replies before the raft store finishes replication, which is not following the common procedure. Also, when an `AcquirePessimisticLock` command meets a lock, it needs to wait for the lock being released in TiKV for a while. And when a lock is released, the awaiting `AcquirePessimisticLock` commands are notified. These are distinctive steps that are not shared by other commands. More dependencies (the lock manager and the deadlock detector) are also introduced.

If more and more commands are not sharing the same procedure and dependencies, the current structure will have no benefit. Instead, the drawbacks become clearer.

The code for a certain command is spread in too many files. Each function handles a part of work of various commands. People are hard to understand what happens about a single command.

And now, different commands have different dependencies. But because all commands share the same procedure, we must pass in the union of all dependencies of all commands. It makes adding commands and dependencies more difficult.

## New structure

Based on the analysis above, the current structure is not flexible enough and should be dropped. Instead, we can change to put logic of a single command together. Common steps can be extracted as methods for reuse. Then, it will be much easier to find out the procedure of each command while still not repeating code.

Steps like acquiring latches and waiting for raft replication cannot finish immediately, so it is appropriate to make every command an `async fn`.

For example, the `AcquirePessimisticLock` command can be like:

```rust
async fn acquire_pessimistic_lock(&self, req: PessimisticLockRequest) -> PessimisticLockResponse {
    let mut resp = PessimisticLockResponse::default();
    let guards = self.lock_keys(req.mutations.iter().map(|m| m.get_key())).await;
    let snapshot = self.get_snapshot(req.get_context()).await;
    
    let mut txn = MvccTxn::new(...);
    let mut locked = None;
    for (mutation, guard) in req.mutations.into_iter().zip(&guards) {
        match txn.acquire_pessimistic_lock(...) {
            Ok(...) => ...,
            Err(KeyIsLocked(lock_info)) => {
                guard.set_lock(lock_info.clone());
                let lock_released = guard.lock_released(); // returns a future which is ready when the lock is released
                locked = Some(lock_info, );
            },
            Err(e) => ...
        }
    }
    
    if let Some((lock_info, lock_released)) = locked {
        drop(guards); // release the latches first
        lock_released.await; // easy to add timeout
        resp.set_errors(...);
    } else {
        if self.cfg.pipelined_pessimistic_lock {
            // write to the raft store asynchronously
            let engine = self.engine.clone();
            self.spawn(async move {
                engine.write(txn.into_modifies()).await;
                drop(guards);
            });
        } else {
            // write to the raft store synchronously
            self.engine.write(txn.into_modifies()).await; 
        }
        ...
    }
    resp
}
```

The goal is to put the whole process of each command inside a single function. Then, people only need to look at one function to learn the process. Code related to transactions will be more understandable.

Moreover, long code paths and jumps are avoided. It's never a problem that dependencies and configurations need be passed through the long path.

## In-memory lock table

Both the latch and the lock manager stores memory locks and notify when the locks are released. For "parallel commit", we also need another memory locking mechanism. It'll be good to have an integrated locking mechanism handling all these requirements.

We can use a concurrent ordered map to build a lock table. We map each raw key to a memory lock. The memory lock contains lock information and waiting lists. Currently we have two kinds of orthogonal waiting list: the latch waiting list and the pessimistic lock waiting list. 

```rust
pub type LockTable = ConcurrentOrderedMap<Vec<u8>, Arc<MemoryLock>>;

pub struct MemoryLock {
    mutex_state: AtomicU64,
    mutex_waiters: ConcurrentList<Notify>
    lock_info: Mutex<Option<LockInfo>>,
    pessimistic_waiters: ConcurrentList<Notify>,
}
```

Both the original latches and lock manager can be implemented with this memory lock.

For the original latch usage, the lock serves as an asynchronous mutex. It can return a future that outputs a guard. The guard can be used to modify the data in the memory lock. When the guard is dropped, other tasks waiting for the lock are notified.

For the lock manager usage, it provides the functionality to add waiters and modify the lock information. When `AcquirePessimisticLock` meets a lock, it adds itself to the waiting list and stores the lock information. When the lock is cleared, the waiters are all notified.

When a guard is dropped, if neither a lock nor any waiter is in the lock, we can remove the key from the map to save memory.

### Parallel commit

The "parallel commit" feature can be also implemented with this lock table. During prewrite, the lock is written to the memory lock before it is sent to the raft store. Before any read request start, we read the lock info in the memory lock. If the `min_commit_ts` recorded in the lock is smaller than the snapshot time stamp, we can return a locked error directly.

```rust
async fn prewrite(&self, req: PrewriteRequest) -> PrewriteResponse {
    ...
    for (lock, guard) in ... {
        // read max_read_ts and set lock atomically
        guard.set_lock(lock, |lock| lock.min_commit_ts = self.max_read_ts() + 1);
    }
    ...
}

async fn get(&self, req: GetRequest) -> GetResponse {
    ...
    if let Err(lock) = self.read_check_key(req.get_key(), req.version.into()) {
        ...
    }
    ...
}

fn read_check_key(&self, key: &[u8], ts: TimeStamp) -> Result<(), LockInfo> {
    self.update_max_read_ts(ts);
    if let Some(lock) = self.lock_table.get(key) {
        let lock_info = lock.lock_info.lock().unwrap();
        if ... {
            return Err(lock_info.clone());
        }
    }
    Ok(())
}

fn read_check_range(&self, start_key: &[u8], end_key: &[u8], ts: TimeStamp) -> Result<(), LockInfo> {
    self.update_max_read_ts(ts);
    if let Some((key, lock)) = self.lock_table.lower_bound(start_key) {
        if key < end_key {
            let lock_info = lock.lock_info.lock().unwrap();
            if ... {
                return Err(lock_info.clone());
            }
        }
    }
    Ok(())
}
```

