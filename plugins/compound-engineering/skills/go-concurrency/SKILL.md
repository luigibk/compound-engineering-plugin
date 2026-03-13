---
name: go-concurrency
description: Goroutines, channels, sync primitives, and context propagation for safe concurrent Go programs. Use when writing concurrent Go code, reviewing goroutine usage, designing worker pools, or debugging race conditions.
---

<objective>
Write and review concurrent Go code using goroutines, channels, sync primitives, and context.Context correctly. Covers the standard library sync package and golang.org/x/sync for real-world concurrency patterns. Target Go 1.21+.
</objective>

<essential_principles>
## Core Concurrency Philosophy

"Do not communicate by sharing memory; instead, share memory by communicating." — Go team

**Go concurrency is built on three pillars:**
1. **Goroutines** — cheap cooperative threads; Go manages the scheduler
2. **Channels** — typed message queues that transfer ownership between goroutines
3. **sync package** — mutexes, wait groups, and once for cases where shared memory is clearest

**Every goroutine needs an exit path.** A goroutine that cannot be stopped is a leak. Before launching any goroutine, answer: *how and when does this goroutine end?*

**context.Context is the cancellation contract.** Any blocking operation that may take time (network, disk, sleep, wait) must accept a `context.Context` and honor cancellation.
</essential_principles>

<intake>
What are you working on?

1. **Goroutines** — launching, lifecycle management, preventing leaks, worker pools
2. **Channels** — direction, select, fan-out/fan-in, done patterns
3. **sync** — Mutex, RWMutex, WaitGroup, Once, Map
4. **context** — propagation, cancellation, timeouts, values
5. **errgroup** — goroutines with error propagation (golang.org/x/sync/errgroup)
6. **Race detection** — finding and fixing data races
7. **Code Review** — review concurrent Go code for correctness
8. **General Guidance** — choose the right primitive for a given problem

**Specify a number or describe your task.**
</intake>

<routing>

| Response | Reference to Read |
|----------|-------------------|
| 1, goroutine, leak, worker, pool | [goroutines.md](./references/goroutines.md) |
| 2, channel, select, fan | [channels.md](./references/channels.md) |
| 3, mutex, rwmutex, waitgroup, once, sync | [sync.md](./references/sync.md) |
| 4, context, cancel, timeout | [sync.md](./references/sync.md) |
| 5, errgroup, semaphore | [sync.md](./references/sync.md) |
| 6, race, data race | [goroutines.md](./references/goroutines.md) |
| 7, review | Read all references, then review code for concurrency issues |
| 8, general | Read relevant references based on the primitive described |

**After reading relevant references, apply the patterns to the user's code.**
</routing>

<quick_reference>
## Goroutine Lifecycle Rule

Every goroutine must have a clear termination path. The three patterns:

```go
// 1. Done channel: goroutine exits when done is closed
go func() {
    for {
        select {
        case <-done:
            return
        case work := <-jobs:
            process(work)
        }
    }
}()

// 2. Context cancellation: goroutine exits when ctx is cancelled
go func(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        case work := <-jobs:
            process(work)
        }
    }
}(ctx)

// 3. Channel close: goroutine exits when jobs channel is closed
go func() {
    for work := range jobs {  // range exits when channel is closed
        process(work)
    }
}()
```

## Channel Direction Types

Enforce direction in function signatures to prevent misuse:

```go
func producer(out chan<- int) { out <- 42 }   // send-only
func consumer(in <-chan int)  { v := <-in }   // receive-only
func pipeline(in <-chan int, out chan<- int) { out <- <-in + 1 }
```

## context.Context Contract

```go
// First argument, always named ctx, never stored in a struct
func GetUser(ctx context.Context, id int64) (*User, error)

// Check cancellation in long operations
select {
case <-ctx.Done():
    return nil, ctx.Err()
default:
    // proceed
}
```

## go test -race

Run all concurrent code with the race detector before shipping:

```bash
go test -race ./...
go build -race ./cmd/server  # instrument the binary itself
```
</quick_reference>

<reference_index>
## Domain Knowledge

| File | Topics |
|------|--------|
| [goroutines.md](./references/goroutines.md) | Lifecycle, leak detection, worker pools, goroutine ownership |
| [channels.md](./references/channels.md) | Direction, select, fan-out/fan-in, pipeline, done patterns |
| [sync.md](./references/sync.md) | Mutex, RWMutex, WaitGroup, Once, errgroup, semaphore, context |
</reference_index>

<success_criteria>
Concurrent Go code is correct when:
- Every goroutine has a documented exit path (context, done channel, or range-close)
- Channel directions are typed in all function signatures (`chan<-`, `<-chan`)
- `sync.Mutex` values are never copied (passed by pointer, not by value)
- `sync.WaitGroup.Add` is called before the goroutine is launched, not inside it
- All blocking operations accept and respect `context.Context`
- `go test -race ./...` passes with no data races
- `errgroup.Group` is used instead of WaitGroup when goroutines can fail
- No goroutine is leaked: all goroutines exit within the lifetime of their owner
</success_criteria>
