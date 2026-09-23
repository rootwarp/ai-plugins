---
description: 'Rust concurrency: ownership as race prevention, Send and Sync, Arc/Mutex and channel patterns, async runtimes. Use when writing threaded or async code, or sharing state across tasks.'
name: rust-concurrency
---

# Concurrency

## Ownership as Race Prevention

Rust's ownership system prevents data races at compile time. Understanding these guarantees is
essential:

- **Ownership**: Only one owner at a time; no concurrent access by default
- **Borrowing**: Multiple `&T` (shared) OR one `&mut T` (exclusive), never both
- **Lifetimes**: Compiler ensures references don't outlive data

```rust
// Compile error: cannot borrow as mutable while borrowed as immutable
let mut data = vec![1, 2, 3];
let reference = &data[0];
data.push(4); // Error!
println!("{}", reference);
```

## Send and Sync Traits

These marker traits define thread safety:

- **`Send`**: Type can be transferred to another thread
- **`Sync`**: Type can be shared between threads (`&T` is `Send`)

### Common Non-Send Types

| Type | Why Not Send |
|------|--------------|
| `Rc<T>` | Non-atomic reference counting |
| `RefCell<T>` | Non-thread-safe interior mutability |
| `*const T`, `*mut T` | Raw pointers have no safety guarantees |
| `MutexGuard<'_, T>` | Must be unlocked on the thread that locked it |
| Types containing any of the above | `Send` is an auto trait; one non-`Send` field propagates |

```rust
use std::rc::Rc;
use std::sync::Arc;

// Compile error: Rc is not Send
let rc = Rc::new(5);
std::thread::spawn(move || println!("{}", rc)); // Error!

// Fix: use Arc for thread-safe reference counting
let arc = Arc::new(5);
std::thread::spawn(move || println!("{}", arc)); // OK
```

## Shared State Patterns

### Arc<Mutex<T>> for Shared Mutable State

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        let mut num = counter.lock().unwrap();
        *num += 1;
    });
    handles.push(handle);
}

for handle in handles {
    handle.join().unwrap();
}

println!("Result: {}", *counter.lock().unwrap());
```

### RwLock for Read-Heavy Workloads

```rust
use std::collections::HashMap;
use std::sync::{Arc, RwLock};

let cache: Arc<RwLock<HashMap<String, u64>>> = Arc::new(RwLock::new(HashMap::new()));

// Multiple readers allowed; copy the value out so the read guard drops at the end of the scope
let cached = {
    let map = cache.read().unwrap();
    map.get("answer").copied()
};

// Exclusive write access; never call `write()` while this thread still holds a read guard
// (it deadlocks or panics)
if cached.is_none() {
    cache.write().unwrap().insert("answer".to_owned(), 42);
}
```

### Atomic Operations

Use atomics for simple shared state:

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

let counter = Arc::new(AtomicUsize::new(0));

// Increment
counter.fetch_add(1, Ordering::SeqCst);

// Load
let value = counter.load(Ordering::SeqCst);
```

**Memory Ordering Guidelines**:
- `Ordering::SeqCst`: Safe default; sequential consistency
- `Ordering::Relaxed`: Only for independent counters/flags
- `Ordering::Acquire`/`Release`: For synchronization pairs
- When in doubt, use `SeqCst`

## Message Passing

### Standard Library mpsc

```rust
use std::sync::mpsc;
use std::thread;

let (tx, rx) = mpsc::channel();

// Producer
thread::spawn(move || {
    for i in 0..10 {
        tx.send(i).unwrap();
    }
});

// Consumer
for received in rx {
    println!("Got: {}", received);
}
```

For backpressure, `mpsc::sync_channel(n)` gives a bounded channel. Since Rust 1.67 std `mpsc`
uses crossbeam's implementation, so performance is not a reason to switch.

### Crossbeam Channels

Reach for `crossbeam-channel` when you need `select!` across channels or multi-consumer (MPMC)
receivers:

```rust
use crossbeam_channel::{bounded, select, unbounded};

// Bounded channel (backpressure)
let (tx, rx) = bounded::<i32>(10);

// Select across multiple channels
select! {
    recv(rx1) -> msg => handle_msg1(msg),
    recv(rx2) -> msg => handle_msg2(msg),
    default => println!("No message available"),
}
```

## Async Considerations

### Pick the Mutex by Whether the Guard Crosses an `.await`

Never hold a `std::sync::Mutex` guard across an `.await` point. When the critical section is
short and contains no `.await`, `std::sync::Mutex` is the right choice even in async code (it is
faster, and this is Tokio's own guidance); use `tokio::sync::Mutex` only when the guard must
live across an `.await`:

```rust
// Bad: std::sync::Mutex guard held across .await. The guard is !Send, so the future is !Send
// and `tokio::spawn` rejects it; on a current-thread runtime another task locking the same
// mutex deadlocks the worker.
async fn bad_example(mutex: Arc<std::sync::Mutex<Data>>) {
    let guard = mutex.lock().unwrap();
    some_async_operation().await;
    drop(guard);
}

// Best: finish the critical section, drop the guard, then await
async fn good_example(mutex: Arc<std::sync::Mutex<Data>>) {
    let input = {
        let guard = mutex.lock().unwrap();
        guard.snapshot()
    };
    some_async_operation(input).await;
}

// When the guard genuinely must live across the await: tokio::sync::Mutex
async fn held_across_await(mutex: Arc<tokio::sync::Mutex<Data>>) {
    let mut guard = mutex.lock().await;
    guard.value = some_async_operation().await;
}
```

Lock results: `lock().unwrap()` (or `.expect("reason")`) is the accepted idiom. A poisoned lock
means another thread already panicked mid-update; propagating that panic is usually right.
Recover with `PoisonError::into_inner` only when the protected data is still valid after a
partial update.

### Async-Aware Primitives

```rust
use tokio::sync::{Mutex, RwLock, Semaphore, mpsc, oneshot};

// Async mutex
let mutex = Mutex::new(data);
let guard = mutex.lock().await;

// Async channel
let (tx, mut rx) = mpsc::channel(100);
tx.send(value).await?;

// One-shot for single response; `send` returns `Err(value)` if the receiver was dropped,
// which is not an error type, so handle it instead of using `?`
let (tx, rx) = oneshot::channel();
if tx.send(response).is_err() {
    tracing::debug!("receiver dropped before the response was sent");
}
let result = rx.await?;

// Semaphore for limiting concurrency
let semaphore = Arc::new(Semaphore::new(10));
let permit = semaphore.acquire().await?;
```

### Blocking and CPU-Bound Work

Never block a runtime worker (sync I/O, `std::thread::sleep`, long computation). Pick by the
kind of work:

| Work | Use |
|------|-----|
| Blocking sync I/O or FFI call | `tokio::task::spawn_blocking` |
| Occasional short CPU burst | `spawn_blocking` |
| Sustained or parallel CPU work | `rayon` (or a dedicated thread pool), result sent back over a `oneshot` |

```rust
use tokio::task;

async fn process_data(data: Vec<u8>) -> Result<Output> {
    // Move blocking work off the async worker threads
    let result = task::spawn_blocking(move || expensive_computation(&data)).await?;
    Ok(result)
}
```

### Spawning and Joining Tasks

- Keep the `JoinHandle` of every spawned task, or use `tokio::task::JoinSet` for a group; a
  detached task's panic and errors vanish silently
- `JoinSet` aborts its remaining tasks when dropped, which gives structured shutdown
- Handle `JoinError` (the task panicked or was cancelled) instead of `unwrap`ping it

```rust
use tokio::task::JoinSet;

let mut set = JoinSet::new();
for url in urls {
    set.spawn(fetch(url));
}
while let Some(joined) = set.join_next().await {
    match joined {
        Ok(Ok(body)) => handle(body),
        Ok(Err(e)) => tracing::warn!(error = %e, "fetch failed"),
        Err(e) => tracing::error!(error = %e, "task panicked or was cancelled"),
    }
}
```

### Cancellation Safety

A future is cancelled by being dropped: `tokio::select!` drops the losing branches, and
`tokio::time::timeout` drops the inner future. Any work the dropped future had done but not
committed is lost.

- In a `select!` loop, only use cancel-safe branches (`mpsc::Receiver::recv`,
  `broadcast::Receiver::recv`, `TcpListener::accept`, `sleep`); tokio documents cancel safety on
  each method
- Not cancel-safe: `AsyncReadExt::read_exact`, `AsyncWriteExt::write_all`, and any hand-written
  future that performs several steps before completing
- For a non-cancel-safe operation, pin it outside the loop and poll `&mut fut` in `select!`, or
  move it into its own task
- Use a `tokio_util::sync::CancellationToken` for cooperative shutdown instead of abort

## Deadlock Patterns

### Lock Ordering

Always acquire locks in a consistent order:

```rust
// Bug: potential deadlock
// Thread 1: lock A, then B
// Thread 2: lock B, then A

// Fix: define a global lock order
fn transfer(from: &Account, to: &Account, amount: u64) {
    // Always lock lower ID first
    let (first, second) = if from.id < to.id {
        (from, to)
    } else {
        (to, from)
    };

    let _guard1 = first.balance.lock().unwrap();
    let _guard2 = second.balance.lock().unwrap();
    // ... perform transfer
}
```

### Self-Deadlock with Non-Reentrant Locks

```rust
// Bug: Mutex is not reentrant
fn outer(mutex: &Mutex<Data>) {
    let _guard = mutex.lock().unwrap();
    inner(mutex); // Deadlock!
}

fn inner(mutex: &Mutex<Data>) {
    let _guard = mutex.lock().unwrap(); // Blocks forever
}

// Fix: restructure so the lock is taken once, e.g. `outer` locks and passes `&mut Data` to an
// `inner_locked(data: &mut Data)` helper. (`parking_lot::ReentrantMutex` only hands out `&T`,
// so it does not allow nested mutation without an extra `RefCell`.)
```

### Async Deadlock from Blocking Waits

A short `std::sync::Mutex` section with no `.await` is fine in async code (see above). The bug
is a *long* blocking wait on a runtime worker: blocking I/O, `std::thread::sleep`, a `Condvar`
wait, or a lock whose holder is itself waiting on an async task.

```rust
// Bug: blocks the worker thread; tasks it should be running (possibly the one that would
// release `rx`'s sender) can starve
async fn bad(rx: std::sync::mpsc::Receiver<Job>) {
    let job = rx.recv().unwrap();
    run(job).await;
}

// Fix: use the async primitive, or move the blocking wait to `spawn_blocking`
async fn good(mut rx: tokio::sync::mpsc::Receiver<Job>) {
    while let Some(job) = rx.recv().await {
        run(job).await;
    }
}
```

## Common Anti-Patterns

### Holding Locks Too Long

```rust
// Bad: lock held during I/O
fn update_and_save(cache: &Mutex<Cache>, key: &str) {
    let mut guard = cache.lock().unwrap();
    guard.insert(key, compute_value());
    save_to_disk(&guard); // Slow I/O while holding lock!
}

// Good: minimize lock scope
fn update_and_save(cache: &Mutex<Cache>, key: &str) {
    let value = compute_value();
    let snapshot = {
        let mut guard = cache.lock().unwrap();
        guard.insert(key, value);
        guard.clone()
    };
    save_to_disk(&snapshot);
}
```

### Arc<Mutex<Vec<T>>> for Append-Only

```rust
// Overkill for append-only log
let log = Arc::new(Mutex::new(Vec::new()));

// Better: use a channel or lock-free structure
let (tx, rx) = crossbeam_channel::unbounded();
tx.send(entry)?;
```

---

## Concurrency Review Checklist

Complements (does not replace) the main checklist in the `rust-code-style` skill. When the
changes touch threads, async, or shared state, also verify:

- [ ] `Arc` used instead of `Rc` for shared ownership across threads
- [ ] `tokio::sync::Mutex` wherever a guard lives across `.await`; `std::sync::Mutex` only for
      short, await-free critical sections
- [ ] No `std::sync::Mutex` held across `.await` points
- [ ] Blocking calls use `spawn_blocking`; sustained CPU work uses `rayon` or a dedicated pool
- [ ] Spawned tasks are joined (`JoinHandle`/`JoinSet`), not silently detached
- [ ] Every `select!` branch is cancel-safe, or the non-cancel-safe future is pinned outside
- [ ] Lock ordering is consistent to prevent deadlocks
- [ ] Lock scopes are minimized (no I/O while holding locks)
- [ ] Atomic operations use appropriate memory ordering (`SeqCst` if unsure)
- [ ] Channels preferred over shared state when possible
- [ ] Lock results use `.unwrap()`/`.expect("reason")` (poison propagates the earlier panic);
      `into_inner` recovery only where partially updated data is still valid
- [ ] Thread-safety traits (`Send`/`Sync`) are satisfied
