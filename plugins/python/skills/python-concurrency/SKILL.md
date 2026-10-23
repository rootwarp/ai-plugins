---
description: 'Python concurrency: asyncio, threads, processes, the effects of the GIL, and cancellation. Use when you write async or concurrent code, or share state between tasks.'
name: python-concurrency
---

# Concurrency

## Select a Model

| Work | Default (GIL) build | Notes |
|------|---------------------|-------|
| Many concurrent I/O waits | `asyncio` | When the stack is already async |
| Blocking I/O, few tasks | `ThreadPoolExecutor` | Libraries without async support |
| CPU-bound pure Python | `ProcessPoolExecutor` | Arguments and functions must pickle |

Two newer options change the CPU-bound row:

- **Free-threaded builds** (PEP 703, `python3.14t`): These builds are experimental in 3.13. They
  have official support since 3.14. Threads run in parallel, so they help CPU-bound work. But
  each C extension that you use must support the build. Code that used the GIL for safety must
  use explicit locks. To check the build, use `sys._is_gil_enabled()`.
- **Subinterpreters** (3.14): `concurrent.futures.InterpreterPoolExecutor` gives each worker its
  own GIL. For CPU-bound pure-Python code, a subinterpreter uses fewer resources than a process.

Use the `concurrent.futures` executors as context managers. The `with` statement shuts down the
executor and waits for it. Do not manage `Thread` or `multiprocessing.Pool` objects manually.

For a process pool, put the entry code in an `if __name__ == "__main__":` block. The spawn start
method (macOS, Windows) and the forkserver start method (the Linux default since 3.14) import the
main module again in each worker. Define worker functions at module level, so that they pickle.

## Threads

- Use `threading.Lock` to protect shared mutable state. Use `RLock` only when the same thread
  enters the lock again.
- To send work between threads, use `queue.Queue`. Do not use shared lists.
- Limit concurrency with the `max_workers` value of an executor or with a `threading.Semaphore`.
- Give each long-running worker a stop signal (`threading.Event`, or a sentinel on the queue).
  At shutdown, join the workers.

## asyncio

Races occur at `await` points. A task reads a value, then awaits, then writes a value that
comes from the read. Another task can run between these steps. Hold an `asyncio.Lock` for the
full read-await-write sequence.

- Run concurrent tasks in an `asyncio.TaskGroup` (3.11+). Do not use `gather`, and do not let
  tasks append to a shared list. Read the results from the tasks, in input order (see the example
  below). On the first failure, the group cancels the other tasks and raises an
  `ExceptionGroup`. Handle it with `except*`.
- In async code, use `asyncio.Lock`, `Semaphore`, `Event`, and `Queue`. A `threading.Lock` that
  is held across an `await` blocks the event loop.
- Move blocking calls (file I/O, CPU work, sync clients) out of the loop with
  `asyncio.to_thread(...)`.
- Keep a reference to each `asyncio.create_task(...)` result that you do not await. The loop
  keeps only weak references to tasks. A better method is to run background tasks in a
  `TaskGroup` that you own.
- Start the program with `asyncio.run(main())`. At exit, it cancels the tasks that remain.

```python
async def fetch_all(urls: list[str]) -> list[bytes]:
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(fetch(url)) for url in urls]
    return [task.result() for task in tasks]
```

## Cancellation and Timeouts

A cancellation comes as `asyncio.CancelledError` at the next `await`. Do the cleanup and raise
the error again. If you swallow the error, `TaskGroup`, `asyncio.timeout()`, and shutdown stop
working correctly.

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

- `except Exception:` does not catch `CancelledError` (3.8+). But `except BaseException:` and a
  bare `except:` catch it. In these handlers, raise it again.
- To set a time limit on a block of awaits, use `async with asyncio.timeout(5):` (3.11+). It
  raises `TimeoutError`.
- Use `asyncio.shield()` only for work that must complete, for example a commit. The caller is
  still cancelled.

## Check-Then-Act and Lazy Initialization

- Hold one lock for both the check and the action
  (`if key not in cache: cache[key] = compute()`). `cache.setdefault(key, compute())` is atomic,
  but it runs `compute()` on each call.
- For a cached value for the full process, use `functools.cache` on a function. Do not write a
  singleton manually. If you must use a lazy singleton, use double-checked locking.
