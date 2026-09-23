---
description: 'Python concurrency: asyncio, threads, processes, GIL implications, cancellation. Use when writing async or concurrent code, or sharing state across tasks.'
name: python-concurrency
---

# Concurrency

## GIL Implications

On the default (GIL) build of CPython:

- **Threading helps for**: I/O-bound tasks (network, disk, sleep)
- **Threading doesn't help for**: CPU-bound tasks (computation)
- **Use multiprocessing for**: CPU-bound parallelism

Two newer options change that trade-off:

- **Free-threaded builds** (PEP 703; `python3.14t`, officially supported since 3.14) run threads
  in parallel, so threads can help CPU-bound work. C extensions must support it, and code that
  relied on the GIL for safety needs explicit locks. Check with `sys._is_gil_enabled()`.
- **Subinterpreters** (3.14): `concurrent.futures.InterpreterPoolExecutor` gives each worker its
  own GIL, a lighter-weight alternative to processes for CPU-bound pure-Python code.

```python
# Good for I/O-bound: threading
import threading

import httpx

def fetch_url(url: str) -> bytes:
    return httpx.get(url, timeout=10).content

threads = [threading.Thread(target=fetch_url, args=(url,)) for url in urls]
for t in threads:
    t.start()
for t in threads:
    t.join()
```

```python
# Good for CPU-bound: multiprocessing
from multiprocessing import Pool

def compute(x: int) -> int:
    return expensive_calculation(x)

def main() -> None:
    with Pool() as pool:
        results = pool.map(compute, range(1000))

# Required: the spawn (macOS, Windows) and forkserver (Linux default since 3.14) start
# methods re-import this module in every worker
if __name__ == "__main__":
    main()
```

## Threading Primitives

### Lock (Mutex)

```python
import threading

class Counter:
    def __init__(self) -> None:
        self._value = 0
        self._lock = threading.Lock()

    def increment(self) -> None:
        with self._lock:
            self._value += 1

    def value(self) -> int:
        with self._lock:
            return self._value
```

### RLock (Reentrant Lock)

Use when the same thread may need to acquire the lock multiple times:

```python
import threading

class RecursiveCounter:
    def __init__(self) -> None:
        self._value = 0
        self._lock = threading.RLock()

    def increment(self) -> None:
        with self._lock:
            self._value += 1

    def increment_twice(self) -> None:
        with self._lock:
            self.increment()  # Can acquire lock again
            self.increment()
```

### Semaphore for Limiting Concurrency

```python
import threading

import httpx

# Limit concurrent connections
connection_semaphore = threading.Semaphore(10)

def make_request(url: str) -> httpx.Response:
    with connection_semaphore:
        return httpx.get(url, timeout=10)
```

### Event for Signaling

```python
import threading

shutdown_event = threading.Event()

def worker() -> None:
    while not shutdown_event.is_set():
        process_work()
        shutdown_event.wait(timeout=1.0)

# Signal shutdown
shutdown_event.set()
```

### Condition for Complex Coordination

```python
import threading
from collections import deque

class BoundedQueue:
    def __init__(self, maxsize: int) -> None:
        self._queue: deque[object] = deque()
        self._maxsize = maxsize
        self._condition = threading.Condition()

    def put(self, item: object) -> None:
        with self._condition:
            while len(self._queue) >= self._maxsize:
                self._condition.wait()
            self._queue.append(item)
            self._condition.notify()

    def get(self) -> object:
        with self._condition:
            while not self._queue:
                self._condition.wait()
            item = self._queue.popleft()
            self._condition.notify()
            return item
```

## Thread-Safe Structures

### queue.Queue

The standard library provides thread-safe queues:

```python
import queue
import threading

work_queue: queue.Queue[Task | None] = queue.Queue()

def worker() -> None:
    while True:
        task = work_queue.get()
        if task is None:  # Shutdown signal
            break
        process(task)
        work_queue.task_done()

# Start workers
threads = [threading.Thread(target=worker) for _ in range(4)]
for t in threads:
    t.start()

# Add work
for task in tasks:
    work_queue.put(task)

# Wait for completion
work_queue.join()

# Shutdown workers
for _ in threads:
    work_queue.put(None)
for t in threads:
    t.join()
```

### Protected Collections Pattern

```python
import threading
from typing import Generic, TypeVar

K = TypeVar("K")
V = TypeVar("V")

# On 3.12+: class ThreadSafeDict[K, V]: (PEP 695) and drop the TypeVars
class ThreadSafeDict(Generic[K, V]):
    def __init__(self) -> None:
        self._data: dict[K, V] = {}
        self._lock = threading.RLock()

    def get(self, key: K, default: V | None = None) -> V | None:
        with self._lock:
            return self._data.get(key, default)

    def set(self, key: K, value: V) -> None:
        with self._lock:
            self._data[key] = value

    def setdefault(self, key: K, value: V) -> V:
        with self._lock:
            return self._data.setdefault(key, value)
```

## Asyncio Race Conditions

### Coroutine Interleaving

Race conditions occur at `await` points, not between them:

```python
import asyncio

# Bug: race condition between await points
class AsyncCounter:
    def __init__(self) -> None:
        self.value = 0

    async def increment(self) -> None:
        current = self.value
        await asyncio.sleep(0)  # Yields control!
        self.value = current + 1  # Race condition

# Fix: use asyncio.Lock
class SafeAsyncCounter:
    def __init__(self) -> None:
        self.value = 0
        self._lock = asyncio.Lock()

    async def increment(self) -> None:
        async with self._lock:
            current = self.value
            await asyncio.sleep(0)
            self.value = current + 1
```

### Collecting Results from Tasks

`list.append` between awaits is not a race in asyncio (one thread, no interleaving), but a
shared module-level list is unordered, leaks across calls, and loses errors. Return results from
the tasks instead:

```python
import asyncio

# Avoid: shared mutable state
results: list[int] = []

async def fetch_and_append(url: str) -> None:
    results.append(await fetch(url))  # completion order, global state

# Prefer: TaskGroup keeps input order and propagates errors
async def fetch_all(urls: list[str]) -> list[int]:
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(fetch(url)) for url in urls]
    return [task.result() for task in tasks]
```

### Asyncio Primitives

```python
import asyncio

# Lock
lock = asyncio.Lock()
async with lock:
    await critical_section()

# Semaphore
semaphore = asyncio.Semaphore(10)
async with semaphore:
    await limited_operation()

# Event: one task waits, another calls event.set()
event = asyncio.Event()
await event.wait()

# Queue
queue: asyncio.Queue[Job] = asyncio.Queue()
await queue.put(job)
job = await queue.get()
```

## Cancellation and Timeouts

Cancellation is delivered as `asyncio.CancelledError` at the next `await`. Clean up and let it
propagate; swallowing it breaks `TaskGroup`, `asyncio.timeout()`, and shutdown.

```python
import asyncio

async def worker(queue: asyncio.Queue[Job]) -> None:
    while True:
        job = await queue.get()
        try:
            await process(job)
        except asyncio.CancelledError:
            queue.put_nowait(job)  # hand the job back, then re-raise
            raise
        finally:
            queue.task_done()

# Bound a whole block of awaits (3.11+); raises TimeoutError
async def fetch_with_deadline(url: str) -> bytes:
    async with asyncio.timeout(5):
        return await fetch(url)
```

- Never `except BaseException:` or bare `except:` around awaits without re-raising
  `CancelledError`; `except Exception:` does not catch it (3.8+)
- Use `asyncio.shield()` only for work that must finish (e.g. a commit); the caller is still
  cancelled
- Cancel background tasks you own on shutdown, or run them inside a `TaskGroup`
- Start the program with `asyncio.run(main())`, which cancels leftover tasks on exit

## concurrent.futures

### ThreadPoolExecutor

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

import httpx

def fetch(url: str) -> httpx.Response:
    return httpx.get(url, timeout=10)

with ThreadPoolExecutor(max_workers=10) as executor:
    # Submit all tasks
    future_to_url = {
        executor.submit(fetch, url): url for url in urls
    }

    # Process as completed
    for future in as_completed(future_to_url):
        url = future_to_url[future]
        try:
            response = future.result()
        except Exception as e:
            print(f"Error fetching {url}: {e}")
```

### ProcessPoolExecutor

```python
from concurrent.futures import ProcessPoolExecutor

def compute(x: int) -> int:
    return expensive_calculation(x)

def main() -> None:
    # Functions and arguments must be picklable (define them at module level)
    with ProcessPoolExecutor() as executor:
        results = list(executor.map(compute, range(1000)))

if __name__ == "__main__":  # required for spawn/forkserver start methods
    main()
```

Both executors are context managers; `with` calls `shutdown(wait=True)` on exit, so no extra
wrapper is needed.

## Common Anti-Patterns

### Check-Then-Act Race

```python
# Bug: race between check and act
if key not in cache:
    cache[key] = compute()  # Another thread may have set it

# Misleading: setdefault is atomic, but compute() still runs every time
value = cache.setdefault(key, compute())

# Fix: hold the lock across check and act
def get_or_compute(key: str) -> Value:
    with cache_lock:
        if key not in cache:
            cache[key] = compute()
        return cache[key]
```

For a process-wide cached value, prefer `functools.cache` on a function over hand-written
locking.

### Lazy Initialization Race

```python
# Bug: race in lazy initialization
class Service:
    _instance = None

    @classmethod
    def get_instance(cls) -> "Service":
        if cls._instance is None:
            cls._instance = cls()  # Race condition!
        return cls._instance

# Fix: use threading.Lock
class Service:
    _instance = None
    _lock = threading.Lock()

    @classmethod
    def get_instance(cls) -> "Service":
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:  # Double-check
                    cls._instance = cls()
        return cls._instance
```

### Forgetting Thread-Safety in Async Code

```python
# Bug: using threading.Lock in async code
class BadAsyncCache:
    def __init__(self) -> None:
        self._lock = threading.Lock()  # Wrong!

    async def get(self, key: str) -> str:
        with self._lock:  # Blocks the event loop!
            return await self._fetch(key)

# Fix: use asyncio.Lock
class GoodAsyncCache:
    def __init__(self) -> None:
        self._lock = asyncio.Lock()

    async def get(self, key: str) -> str:
        async with self._lock:
            return await self._fetch(key)
```

### Blocking Calls in Async Functions

```python
import asyncio
from pathlib import Path

# Bug: blocking call in async context
async def bad_read_file(path: str) -> str:
    return open(path).read()  # Blocks event loop!

# Fix: run blocking I/O in a worker thread (asyncio.to_thread, Python 3.9+)
async def good_read_file(path: str) -> str:
    return await asyncio.to_thread(Path(path).read_text)

# Alternative: aiofiles (third-party, also thread-backed; convenient for streaming reads)
import aiofiles

async def read_file_aiofiles(path: str) -> str:
    async with aiofiles.open(path) as f:
        return await f.read()
```

---

## Concurrency Review Checklist

Complements (does not replace) the main checklist in the `python-code-style` skill. When the
changes touch threads, async, multiprocessing, or shared state, also verify:

- [ ] Appropriate concurrency model chosen (threads for I/O; processes, subinterpreters, or a
  free-threaded build for CPU)
- [ ] All shared mutable state protected by locks
- [ ] `threading.Lock` used for threading, `asyncio.Lock` for asyncio
- [ ] No blocking calls in async functions (use `asyncio.to_thread` or `run_in_executor`)
- [ ] `queue.Queue` used for thread communication (not bare lists)
- [ ] Thread/process pools properly shut down (use context managers)
- [ ] Multiprocessing entry code behind `if __name__ == "__main__":`
- [ ] `CancelledError` is re-raised after cleanup, never swallowed
- [ ] No check-then-act patterns without proper locking
- [ ] Lazy initialization is locked (double-checked locking or `functools.cache`)
- [ ] `asyncio.TaskGroup` used for concurrent async tasks (Python 3.11+)
- [ ] Worker threads/processes have clean shutdown mechanism
