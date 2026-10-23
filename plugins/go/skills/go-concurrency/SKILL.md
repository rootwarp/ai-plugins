---
description: 'Go concurrency: goroutines, channels, context, sync primitives, leak and race avoidance. Use when you write concurrent code or share state across goroutines.'
name: go-concurrency
---

# Concurrency

## Race Detector

Run the tests of concurrent code with `go test -race ./...`. Do this in CI also. `-race` needs
cgo and a C compiler. If they are not available, tell the user that the race detector did not
run. Do not remove it silently.

## Goroutine Lifecycle

Each goroutine must have a way to stop. Its owner must have a way to wait for it. Long work in
the goroutine must also obey `ctx`.

```go
func startWorker(ctx context.Context) <-chan struct{} {
    done := make(chan struct{})
    go func() {
        defer close(done)
        ticker := time.NewTicker(time.Second)
        defer ticker.Stop()
        for {
            select {
            case <-ctx.Done():
                return
            case <-ticker.C:
                process(ctx)
            }
        }
    }()
    return done // caller: cancel(); <-done
}
```

Do not put the work in a `select` with an empty `default:`. That loop uses the CPU continuously.
Also, it sees the cancellation only between calls.

### Fan-out

- **Goroutines that can fail:** Use `golang.org/x/sync/errgroup`. It waits for all goroutines,
  returns the first error, and cancels the shared context. It also sets a limit on
  concurrency. Thus you do not need a custom worker pool.

  ```go
  g, ctx := errgroup.WithContext(ctx)
  g.SetLimit(8)
  for _, url := range urls {
      g.Go(func() error { return fetch(ctx, url) })
  }
  return g.Wait()
  ```

- **Goroutines that cannot fail:** Use `wg.Go(func() { ... })` (Go 1.25+). It calls `Add` and
  `Done` for you. If `go.mod` declares an older `go` version, call `wg.Add(1)` before the `go`
  statement. Then call `defer wg.Done()` in the goroutine.
- **Loop variables:** Since Go 1.22, each iteration has its own loop variable. Before 1.22
  (look in `go.mod`), give the loop variable to the goroutine as an argument.

## Channels

- Only the sender closes a channel. A close of a nil or closed channel causes a panic. A send on
  a closed channel also causes a panic.
- In a `select` on many inputs, set a channel to `nil` after it closes. Then its case does not
  fire again. Exit when all the channels are nil.
- A send in a loop must also select on `ctx.Done()`. A bare `out <- v` blocks forever after the
  receiver stops:

  ```go
  select {
  case out <- v:
  case <-ctx.Done():
      return ctx.Err()
  }
  ```

- Buffer size:
  - Use an unbuffered channel for synchronization.
  - Use size 1 for signals that can coalesce.
  - Use a known bound for work queues.

  Large buffers hide backpressure problems. Thus each buffer size must have a reason.

## Shared State

- By default, protect shared maps and slices with a `sync.Mutex`. Use `sync.RWMutex` only if
  measurements show that most access is reads. Use `sync.Map` only for its documented cases:
  - Keys that are written one time and read many times
  - Goroutines that use disjoint sets of keys
- A check followed by an action (`if _, ok := m[k]; !ok { m[k] = compute() }`) is a race. Hold
  one lock for the check and the write. If `compute` is expensive and concurrent callers must
  share one result, use `golang.org/x/sync/singleflight`.
- Locked methods must return copies of internal slices and maps. If a method returns the field,
  the caller gets access without synchronization.
- Use `sync/atomic` types (`atomic.Int64`, `atomic.Bool`) for counters and flags.
- Use `sync.OnceValue` or `sync.OnceValues` for lazy values that the full process shares
  (compiled regexps, lookup tables). Use `sync.Once` for side effects. Make config, clients,
  and repositories in `main`, and give them through constructors. Do not hide them in a global
  that a once function initializes (see the `go-architecture` skill).
