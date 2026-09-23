---
description: 'Go concurrency: goroutines, channels, context, sync primitives, leak and race avoidance. Use when writing concurrent code or sharing state across goroutines.'
name: go-concurrency
---

# Concurrency

## Race Detection

Always test concurrent code with the race detector:

```bash
# Run tests with race detection
go test -race ./...

# Build with race detection for development
go build -race

# Run specific benchmarks with race detection
go test -race -bench=. ./...
```

**CI Integration**: Enable `-race` flag in CI pipelines for all test runs. Race conditions caught in
CI are far cheaper than production bugs.

## Goroutine Lifecycle

### Never Start a Goroutine Without a Shutdown Plan

Every goroutine must have a clear mechanism for termination.

```go
// Bad: goroutine leak - no way to stop
func startWorker() {
    go func() {
        for {
            process()
        }
    }()
}

// Good: context-based cancellation, and the caller can wait for the exit
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
                process(ctx) // long-running work must honor ctx too
            }
        }
    }()
    return done // caller: cancel(); <-done
}
```

Avoid a `select` with an empty `default:` around the work — it busy-spins and only notices
cancellation between calls.

### WaitGroup for Goroutine Completion

Use `WaitGroup.Go` (Go 1.25+); it does the `Add`/`Done` bookkeeping that is easy to misplace:

```go
func processItems(items []Item) {
    var wg sync.WaitGroup
    for _, item := range items {
        wg.Go(func() {
            process(item) // per-iteration variable since Go 1.22
        })
    }
    wg.Wait()
}
```

When `go.mod` declares a `go` version below 1.25, call `wg.Add(1)` before the `go` statement
and `defer wg.Done()` inside it; below 1.22, also pass the loop variable as an argument (see
Loop Variable Capture below).

### errgroup for Goroutines That Can Fail

Prefer `golang.org/x/sync/errgroup` when any goroutine can return an error: it waits for all of
them, returns the first error, cancels the shared context, and can bound concurrency.

```go
func fetchAll(ctx context.Context, urls []string) error {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(8) // at most 8 in flight

    for _, url := range urls {
        g.Go(func() error {
            return fetch(ctx, url)
        })
    }
    return g.Wait()
}
```

### Worker Pool Pattern

```go
func workerPool(ctx context.Context, jobs <-chan Job, numWorkers int) {
    var wg sync.WaitGroup

    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case job, ok := <-jobs:
                    if !ok {
                        return
                    }
                    process(job)
                case <-ctx.Done():
                    return
                }
            }
        }()
    }

    wg.Wait()
}
```

## Channel Patterns

### Close Semantics

- Only the sender should close a channel
- Never close a channel from the receiver side
- Closing a nil channel panics
- Sending on a closed channel panics
- Receiving from a closed channel returns zero value immediately

```go
// Good: sender closes
func producer(out chan<- int) {
    defer close(out)
    for i := 0; i < 10; i++ {
        out <- i
    }
}

// Good: range over channel (stops when closed)
func consumer(in <-chan int) {
    for val := range in {
        process(val)
    }
}
```

### Select for Multiple Channels

```go
func multiplex(ctx context.Context, a, b <-chan int, out chan<- int) {
    for {
        select {
        case v, ok := <-a:
            if !ok {
                a = nil // Disable this case
                break   // leave the select; the nil check below runs
            }
            if !send(ctx, out, v) {
                return
            }
        case v, ok := <-b:
            if !ok {
                b = nil
                break
            }
            if !send(ctx, out, v) {
                return
            }
        case <-ctx.Done():
            return
        }

        if a == nil && b == nil {
            return
        }
    }
}

// send blocks until out accepts v or ctx is cancelled; a bare `out <- v` would ignore ctx.
func send(ctx context.Context, out chan<- int, v int) bool {
    select {
    case out <- v:
        return true
    case <-ctx.Done():
        return false
    }
}
```

### Channel Sizing Guidelines

- **Unbuffered (size 0)**: Synchronization required; sender blocks until receiver ready
- **Size 1**: Signal channels, single handoff
- **Known bounded size**: When producer/consumer rates are predictable
- **Avoid large buffers**: They hide problems; prefer backpressure

```go
// Done signal: unbuffered, closed by the sender to broadcast
done := make(chan struct{})

// Non-blocking "wake up" notification (size 1; extra signals coalesce)
wake := make(chan struct{}, 1)

// Work queue with bounded concurrency
jobs := make(chan Job, numWorkers)
```

## Sync Package

### Mutex Best Practices

```go
type SafeCounter struct {
    mu    sync.Mutex // Zero value is valid, unlocked mutex
    count int
}

func (c *SafeCounter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}
```

### RWMutex for Read-Heavy Workloads

```go
type Cache struct {
    mu    sync.RWMutex
    items map[string]Item
}

func (c *Cache) Get(key string) (Item, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    item, ok := c.items[key]
    return item, ok
}

func (c *Cache) Set(key string, item Item) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = item
}
```

### sync.OnceValue for Lazy One-Time Initialization

Use it for expensive, process-wide values that are genuinely global (compiled regexps, lookup
tables). Application dependencies such as config, clients, and repositories are built in `main`
and passed through constructors instead (see the `go-architecture` skill).

```go
var defaultTable = sync.OnceValue(func() *Table {
    return buildTable() // runs once, on first call
})

func lookup(k string) string {
    return defaultTable().Get(k)
}
```

`sync.OnceValues` does the same for a `(value, error)` pair; plain `sync.Once` remains for
side effects without a result.

### Atomic Operations

Use `sync/atomic` for simple counters and flags:

```go
type Stats struct {
    requests atomic.Int64
    errors   atomic.Int64
}

func (s *Stats) RecordRequest() {
    s.requests.Add(1)
}

func (s *Stats) RecordError() {
    s.errors.Add(1)
}
```

## Common Anti-Patterns

### Loop Variable Capture (Pre-Go 1.22)

Only relevant when `go.mod` declares `go` below 1.22:

```go
// Bug: all goroutines share the same loop variable
for _, item := range items {
    go func() {
        process(item) // Race condition!
    }()
}

// Fix: pass as argument
for _, item := range items {
    go func(it Item) {
        process(it)
    }(item)
}

// Note: Go 1.22+ fixes this with per-iteration loop variables
```

### Shared Map/Slice Without Synchronization

```go
// Bug: concurrent map write
var cache = make(map[string]int)

go func() { cache["a"] = 1 }()
go func() { cache["b"] = 2 }() // Panic: concurrent map writes

// Fix: guard the map with a mutex (the default choice)
var (
    mu    sync.Mutex
    cache = make(map[string]int)
)

go func() { mu.Lock(); cache["a"] = 1; mu.Unlock() }()
go func() { mu.Lock(); cache["b"] = 2; mu.Unlock() }()
```

Reach for `sync.Map` only for its documented cases: keys written once and read many times, or
goroutines working on disjoint key sets.

### Check-Then-Act Race

```go
// Bug: race between check and act
if _, ok := cache[key]; !ok {
    cache[key] = compute() // Another goroutine may have set it
}

// Fix: use sync.Map's LoadOrStore (compute() still runs on every call; when it is expensive,
// hold a mutex around the check and the write, or use golang.org/x/sync/singleflight)
actual, loaded := cache.LoadOrStore(key, compute())
```

### Forgetting to Copy Slices at Boundaries

```go
type Server struct {
    mu      sync.Mutex
    clients []Client
}

// Bug: returns internal slice
func (s *Server) GetClients() []Client {
    s.mu.Lock()
    defer s.mu.Unlock()
    return s.clients // Caller can modify!
}

// Fix: return a copy
func (s *Server) GetClients() []Client {
    s.mu.Lock()
    defer s.mu.Unlock()
    result := make([]Client, len(s.clients))
    copy(result, s.clients)
    return result
}
```

---

## Concurrency Review Checklist

Complements (does not replace) the main checklist in the `go-code-style` skill. When the
changes touch goroutines, channels, or shared state, also verify:

- [ ] All tests pass with `-race` flag
- [ ] Every goroutine has a clear shutdown mechanism (context, done channel, or WaitGroup)
- [ ] Channels are closed only by senders
- [ ] No shared maps/slices accessed without synchronization
- [ ] `WaitGroup.Go` used (Go 1.25+), or `wg.Add` called before the `go` statement
- [ ] Loop variables are not captured in goroutines (or `go.mod` declares Go 1.22+)
- [ ] Mutex/RWMutex used appropriately (RWMutex only for read-heavy workloads)
- [ ] No check-then-act patterns without proper locking
- [ ] Slices/maps copied at API boundaries
- [ ] Channel buffer sizes are justified (prefer unbuffered or size-1)
- [ ] Lazy one-time init uses `sync.OnceValue`/`sync.Once`, not hand-rolled flags; no global
      singletons for injectable dependencies
- [ ] Channel sends inside loops also select on `ctx.Done()`
- [ ] Goroutines that can fail run under `errgroup` (first error cancels the rest)
