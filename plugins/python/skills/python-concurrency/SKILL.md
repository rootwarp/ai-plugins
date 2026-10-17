---
description: 'Python concurrency: asyncio, threads, processes, GIL implications, cancellation. Use when writing async or concurrent code, or sharing state across tasks.'
name: python-concurrency
---

# Concurrency

## Choosing a Model

| Work | Default (GIL) build | Notes |
|------|---------------------|-------|
| Many concurrent I/O waits | `asyncio` | When the stack is already async |
| Blocking I/O, few tasks | `ThreadPoolExecutor` | Libraries without async support |
| CPU-bound pure Python | `ProcessPoolExecutor` | Arguments and functions must pickle |

Two newer options change the CPU-bound row:

- **Free-threaded builds** (PEP 703; `python3.14t`): experimental in 3.13 and officially
  supported since 3.14. Threads run in parallel, so they help CPU-bound work, but every C
  extension in use must support the build, and code that relied on the GIL for safety needs
  explicit locks. Check with `sys._is_gil_enabled()`.
- **Subinterpreters** (3.14): `concurrent.futures.InterpreterPoolExecutor` gives each worker its
  own GIL, which is lighter than a process for CPU-bound pure-Python code.

Prefer the `concurrent.futures` executors as context managers (`with` shuts them down and waits)
over hand-managed `Thread` or `multiprocessing.Pool` objects. Put process-pool entry code behind
`if __name__ == "__main__":` because the spawn (macOS, Windows) and forkserver (Linux default
since 3.14) start methods re-import the main module in every worker, and define worker functions
at module level so they pickle.

## Threads

- Guard shared mutable state with `threading.Lock`; use `RLock` only when the same thread
  re-enters the lock.
- Pass work between threads through `queue.Queue`, not shared lists.
- Bound concurrency with an executor's `max_workers` or a `threading.Semaphore`.
- Give long-running workers a stop signal (`threading.Event`, or a sentinel on the queue) and
  join them on shutdown.

## asyncio

Races happen at `await` points: a read, an `await`, then a write based on the read can interleave
with another task. Hold an `asyncio.Lock` across the whole read-await-write sequence.

- Run concurrent tasks in an `asyncio.TaskGroup` (3.11+) instead of `gather` or tasks appending
  to a shared list, and read results from the tasks (input order, below). The group cancels
  siblings on the first failure and raises an `ExceptionGroup`; handle it with `except*`.
- Use `asyncio.Lock`, `Semaphore`, `Event`, and `Queue` in async code. A `threading.Lock` held
  across an `await` blocks the event loop.
- Move blocking calls (file I/O, CPU work, sync clients) off the loop with
  `asyncio.to_thread(...)`.
- Keep a reference to every `asyncio.create_task(...)` result you don't await, because the loop
  holds tasks only weakly; better, run background tasks inside a `TaskGroup` you own.
- Start the program with `asyncio.run(main())`, which cancels leftover tasks on exit.

```python
async def fetch_all(urls: list[str]) -> list[bytes]:
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(fetch(url)) for url in urls]
    return [task.result() for task in tasks]
```

## Cancellation and Timeouts

Cancellation arrives as `asyncio.CancelledError` at the next `await`. Clean up and re-raise:
swallowing it breaks `TaskGroup`, `asyncio.timeout()`, and shutdown.

```python
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
```

- `except Exception:` does not catch `CancelledError` (3.8+); `except BaseException:` and bare
  `except:` do, so re-raise it there.
- Bound a block of awaits with `async with asyncio.timeout(5):` (3.11+), which raises
  `TimeoutError`.
- Use `asyncio.shield()` only for work that must finish, such as a commit; the caller is still
  cancelled.

## Check-Then-Act and Lazy Initialization

- Hold one lock across the check and the act (`if key not in cache: cache[key] = compute()`).
  `cache.setdefault(key, compute())` is atomic but still runs `compute()` on every call.
- For a process-wide cached value, prefer `functools.cache` on a function over a hand-written
  singleton; when a lazy singleton is unavoidable, use double-checked locking.
