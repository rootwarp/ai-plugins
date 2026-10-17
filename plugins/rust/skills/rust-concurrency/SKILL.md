---
description: 'Rust concurrency: ownership as race prevention, Send and Sync, Arc/Mutex and channel patterns, async runtimes. Use when writing threaded or async code, or sharing state across tasks.'
name: rust-concurrency
---

# Concurrency

The compiler rules out data races. What it does not catch is deadlocks, blocked runtimes, lost
work on cancellation, and silently dropped task errors, so that is where this skill focuses.

## Shared State

- Prefer message passing or ownership transfer to shared state. Reach for `Arc<Mutex<T>>` when
  several tasks really mutate one value. Use `RwLock` only for read-heavy data, and never call
  `write()` while the same thread still holds a read guard (it deadlocks or panics).
- Atomics: `Relaxed` only for independent counters and flags, `Acquire`/`Release` for
  publish/consume pairs, `SeqCst` when unsure.
- Lazy globals use `std::sync::LazyLock` / `OnceLock`, not `lazy_static` or `once_cell`.
- Threads that borrow local data use `std::thread::scope` instead of `Arc`-wrapping it.
- Lock results: `lock().unwrap()` (or `.expect("reason")`) is the accepted idiom. A poisoned
  lock means another thread already panicked mid-update, and propagating that panic is usually
  right. Recover with `PoisonError::into_inner` only when the data is still valid after a
  partial update.
- Keep critical sections short. Compute before locking, copy or clone what you need out, and do
  I/O after the guard drops.

## Channels

- std `mpsc` has used crossbeam's implementation since Rust 1.67, so performance is not a reason
  to switch. Use `mpsc::sync_channel(n)` for backpressure.
- Use `crossbeam-channel` when you need `select!` across channels or multi-consumer receivers.
- An append-only log shared by many producers is a channel, not `Arc<Mutex<Vec<T>>>`.

## Async

### Mutex Choice: Does the Guard Cross an `.await`?

A `std::sync::Mutex` guard is `!Send`. Holding it across `.await` makes the future `!Send`, so
`tokio::spawn` rejects it, and on a current-thread runtime it can deadlock. For short, await-free
critical sections `std::sync::Mutex` is the right choice even in async code (faster, and Tokio's
own guidance). Finish the section in a block, then await:

```rust
let input = {
    let guard = mutex.lock().unwrap();
    guard.snapshot()
};
some_async_operation(input).await;
```

Use `tokio::sync::Mutex` only when the guard really must live across an `.await`.

### Blocking and CPU-Bound Work

Never block a runtime worker with sync I/O, `std::thread::sleep`, a `Condvar` or std-channel
wait, or a long computation. That starves the other tasks on the worker, possibly including the
one that would unblock it.

| Work | Use |
|------|-----|
| Blocking sync I/O or FFI call | `tokio::task::spawn_blocking` |
| Occasional short CPU burst | `spawn_blocking` |
| Sustained or parallel CPU work | `rayon` (or a dedicated pool), result sent back on a `oneshot` |

Waiting on a channel or lock from async code uses the `tokio::sync` primitive
(`mpsc`, `Mutex`, `Semaphore`, …), not the std one.

### Spawned Tasks

- Keep the `JoinHandle` of every spawned task, or use `tokio::task::JoinSet` for a group. A
  detached task's panics and errors vanish silently.
- `JoinSet` aborts its remaining tasks when dropped, which gives structured shutdown.
- Handle `JoinError` (the task panicked or was cancelled) instead of `unwrap`ping it.
- `oneshot::Sender::send` returns `Err(value)` when the receiver is gone. That is not an error
  type, so handle it explicitly rather than with `?`.
- Use a `tokio_util::sync::CancellationToken` for cooperative shutdown instead of `abort`.

### Cancellation Safety

A future is cancelled by being dropped: `tokio::select!` drops the losing branches, and
`tokio::time::timeout` drops the inner future. Work the dropped future had done but not committed
is lost.

- In a `select!` loop, use only cancel-safe branches (`mpsc::Receiver::recv`,
  `broadcast::Receiver::recv`, `TcpListener::accept`, `sleep`). Tokio documents cancel safety on
  each method.
- Not cancel-safe: `AsyncReadExt::read_exact`, `AsyncWriteExt::write_all`, and any hand-written
  future that performs several steps before completing.
- For a non-cancel-safe operation, pin it outside the loop and poll `&mut fut` in `select!`, or
  move it into its own task.

## Deadlocks

- Acquire multiple locks in one global order (e.g. lower account id first).
- `std::sync::Mutex` is not reentrant: a function holding the guard must not call code that
  locks again. Lock once and pass `&mut T` to an inner helper.
  (`parking_lot::ReentrantMutex` only hands out `&T`, so it does not fix nested mutation.)
- In async code, a lock whose holder is itself waiting on an async task deadlocks the worker.
