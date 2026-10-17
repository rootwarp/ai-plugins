---
description: 'Go concurrency: goroutines, channels, context, sync primitives, leak and race avoidance. Use when writing concurrent code or sharing state across goroutines.'
name: go-concurrency
---

# Concurrency

## Race Detector

Run concurrent code's tests with `go test -race ./...`, in CI as well. `-race` needs cgo and a C
compiler. Where those are missing, say the race detector was skipped rather than dropping it
silently.

## Goroutine Lifecycle

Every goroutine needs a way to stop and a way for its owner to wait for it. Long-running work
inside it honors `ctx` too.

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

Don't wrap the work in a `select` with an empty `default:`. It busy-spins and only notices
cancellation between calls.

### Fan-out

- **Goroutines that can fail:** use `golang.org/x/sync/errgroup`. It waits for all of them,
  returns the first error, cancels the shared context, and bounds concurrency, which replaces
  a hand-rolled worker pool:

  ```go
  g, ctx := errgroup.WithContext(ctx)
  g.SetLimit(8)
  for _, url := range urls {
      g.Go(func() error { return fetch(ctx, url) })
  }
  return g.Wait()
  ```

- **Goroutines that can't fail:** use `wg.Go(func() { ... })` (Go 1.25+), which does the
  `Add`/`Done` bookkeeping. When `go.mod` declares an older `go` version, call `wg.Add(1)`
  before the `go` statement and `defer wg.Done()` inside it.
- **Loop variables:** they're per-iteration since Go 1.22. Below 1.22 (check `go.mod`), pass
  the loop variable to the goroutine as an argument.

## Channels

- Only the sender closes a channel. Closing a nil or closed channel panics, and so does sending
  on a closed one.
- In a `select` over several inputs, set a channel to `nil` once it's closed so its case stops
  firing, and exit when all are nil.
- A send inside a loop also selects on `ctx.Done()`. A bare `out <- v` blocks forever once the
  receiver has gone:

  ```go
  select {
  case out <- v:
  case <-ctx.Done():
      return ctx.Err()
  }
  ```

- Sizing: unbuffered for synchronization, size 1 for signals that may coalesce, a known bound
  for work queues. Large buffers hide backpressure problems, so each buffer size needs a reason.

## Shared State

- Guard shared maps and slices with a `sync.Mutex` by default. Use `sync.RWMutex` only for
  measurably read-heavy access. Use `sync.Map` only for its documented cases: keys written
  once and read many times, or goroutines working on disjoint key sets.
- Check-then-act (`if _, ok := m[k]; !ok { m[k] = compute() }`) is a race. Hold one lock across
  the check and the write. When `compute` is expensive and concurrent callers should share one
  result, use `golang.org/x/sync/singleflight`.
- Return copies of internal slices and maps from locked methods. Returning the field hands the
  caller unsynchronized access.
- Use `sync/atomic` types (`atomic.Int64`, `atomic.Bool`) for counters and flags.
- Use `sync.OnceValue` / `sync.OnceValues` for genuinely process-wide lazy values (compiled
  regexps, lookup tables) and `sync.Once` for side effects. Config, clients, and repositories
  are built in `main` and passed through constructors, not hidden behind a once-initialized
  global (see the `go-architecture` skill).
