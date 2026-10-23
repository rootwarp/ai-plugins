---
description: 'Rust concurrency: ownership as race prevention, Send and Sync, Arc/Mutex and channel patterns, and async runtimes. Use when you write threaded or async code, or when you share state across tasks.'
name: rust-concurrency
---

# Concurrency

The compiler prevents data races. It does not find deadlocks, blocked runtimes, work that
cancellation loses, or task errors that are silently dropped. This skill is about those
problems.

## Shared State

- Use message passing or ownership transfer, not shared state, where possible. Use
  `Arc<Mutex<T>>` when several tasks really must change one value. Use `RwLock` only for data
  that is mostly read. Never call `write()` while the same thread holds a read guard, because
  the call deadlocks or panics.
- Atomics: Use `Relaxed` only for independent counters and flags. Use `Acquire`/`Release` for
  publish/consume pairs. If you are not sure, use `SeqCst`.
- For lazy globals, use `std::sync::LazyLock` or `OnceLock`, not `lazy_static` or `once_cell`.
- For threads that borrow local data, use `std::thread::scope`. Do not wrap the data in `Arc`.
- Lock results: `lock().unwrap()` (or `.expect("reason")`) is the accepted idiom. A poisoned
  lock means that another thread panicked during an update. Usually, the correct action is to
  propagate that panic. Use `PoisonError::into_inner` to recover only when the data is still
  valid after a partial update.
- Keep critical sections short. Do the computation before you lock. Copy or clone the data that
  you need out of the lock. Do I/O after the guard drops.

## Channels

- Since Rust 1.67, std `mpsc` uses the implementation of crossbeam. Performance is thus not a
  reason to change to a different channel. Use `mpsc::sync_channel(n)` for backpressure.
- Use `crossbeam-channel` when you need `select!` across channels or receivers with more than
  one consumer.
- For an append-only log that many producers share, use a channel, not `Arc<Mutex<Vec<T>>>`.

## Async

### Mutex Choice: Does the Guard Cross an `.await`?

A `std::sync::Mutex` guard is `!Send`. If you hold it across `.await`, the future becomes
`!Send`. Then `tokio::spawn` rejects the future, and on a current-thread runtime it can
deadlock. For short critical sections without an `.await`, `std::sync::Mutex` is the correct
choice, also in async code. It is faster, and the Tokio guidance recommends it. Finish the
section in a block, then await:

```rust
let input = {
    let guard = mutex.lock().unwrap();
    guard.snapshot()
};
some_async_operation(input).await;
```

Use `tokio::sync::Mutex` only when the guard really must stay alive across an `.await`.

### Blocking and CPU-Bound Work

Never block a runtime worker with sync I/O, `std::thread::sleep`, a wait on a `Condvar` or a std
channel, or a long computation. A blocked worker stops the other tasks on that worker. One of
these tasks can be the task that would unblock it.

| Work | Use |
|------|-----|
| Blocking sync I/O or FFI call | `tokio::task::spawn_blocking` |
| Occasional short CPU burst | `spawn_blocking` |
| Sustained or parallel CPU work | `rayon` (or a dedicated pool), result sent back on a `oneshot` |

When async code waits on a channel or a lock, use the `tokio::sync` primitive (`mpsc`, `Mutex`,
`Semaphore`, …), not the std primitive.

### Spawned Tasks

- Keep the `JoinHandle` of each spawned task, or use `tokio::task::JoinSet` for a group. If a
  task is detached, its panics and errors silently disappear.
- When a `JoinSet` is dropped, it aborts the tasks that remain. This gives a structured
  shutdown.
- Handle `JoinError` (the task panicked or was cancelled). Do not `unwrap` it.
- `oneshot::Sender::send` returns `Err(value)` when the receiver no longer exists. The value is
  not an error type, so handle it explicitly. Do not use `?`.
- For cooperative shutdown, use a `tokio_util::sync::CancellationToken`, not `abort`.

### Cancellation Safety

When a future is dropped, it is cancelled. `tokio::select!` drops the branches that do not
complete first, and `tokio::time::timeout` drops the inner future. If the dropped future did
work but did not commit it, that work is lost.

- In a `select!` loop, use only cancel-safe branches (`mpsc::Receiver::recv`,
  `broadcast::Receiver::recv`, `TcpListener::accept`, `sleep`). The Tokio docs give the cancel
  safety of each method.
- These are not cancel-safe: `AsyncReadExt::read_exact`, `AsyncWriteExt::write_all`, and any
  hand-written future that does several steps before it completes.
- For an operation that is not cancel-safe, pin it outside the loop and poll `&mut fut` in
  `select!`. As an alternative, move the operation into a separate task.

## Deadlocks

- Acquire multiple locks in one global order. For example, lock the account with the lower id
  first.
- `std::sync::Mutex` is not reentrant. A function that holds the guard must not call code that
  locks the mutex again. Lock one time and pass `&mut T` to an inner helper.
  (`parking_lot::ReentrantMutex` gives only `&T`, so it does not solve nested mutation.)
- In async code, if the holder of a lock waits on an async task, the worker deadlocks.
