# Channels: Direction, Select, Fan-out/Fan-in, Pipeline, Done Patterns

Target: Go 1.21+

---

## Channel Fundamentals

A channel is a typed conduit for communication between goroutines. Channels transfer both data and **ownership** — the receiver takes ownership of the sent value.

```go
ch := make(chan int)      // unbuffered: send blocks until receiver is ready
ch := make(chan int, 10)  // buffered: send blocks only when buffer is full
```

### Unbuffered vs Buffered

| | Unbuffered | Buffered |
|---|---|---|
| Send blocks | Until receiver is ready | Until buffer is full |
| Synchronization | Synchronous rendezvous | Asynchronous (up to capacity) |
| Use when | Guaranteed handoff needed | Decoupling producer/consumer rate |

**Default to unbuffered.** Use buffered channels only when you have a measured reason (bursty producers, decoupled rates, preventing goroutine leaks in specific patterns).

---

## Channel Direction

Enforce send/receive direction in function signatures. The compiler prevents misuse:

```go
// Untyped (accepted but imprecise)
func produce(ch chan int) { ch <- 42 }
func consume(ch chan int) { v := <-ch }

// Typed direction (correct — documents intent and prevents bugs)
func produce(out chan<- int) { out <- 42 }   // can only send
func consume(in <-chan int)  { v := <-in }   // can only receive
```

Bidirectional `chan T` is still needed when creating the channel and passing it to both producer and consumer:

```go
func main() {
    ch := make(chan int)       // bidirectional at creation
    go produce(ch)            // passed as chan<- int
    consume(ch)               // passed as <-chan int
}
```

---

## select Statement

`select` is the multiplexing primitive for channels — it waits on whichever case is ready first.

### Basic Select

```go
select {
case msg := <-messages:
    process(msg)
case err := <-errors:
    log.Printf("error: %v", err)
case <-ctx.Done():
    return ctx.Err()
}
```

### Non-blocking Check (default clause)

```go
select {
case msg := <-ch:
    process(msg)
default:
    // ch is empty; do something else
}
```

**Use `default` sparingly.** A tight loop with `default` becomes a busy-wait that pegs the CPU. Always include a sleep or yield when polling:

```go
// WRONG: spins at 100% CPU
for {
    select {
    case msg := <-ch:
        process(msg)
    default:
        // burn CPU
    }
}

// RIGHT: use a ticker or block properly
for {
    select {
    case msg := <-ch:
        process(msg)
    case <-time.After(100 * time.Millisecond):
        // periodic polling with a pause
    }
}
```

### Timeout

```go
select {
case result := <-resultCh:
    return result, nil
case <-time.After(5 * time.Second):
    return nil, ErrTimeout
case <-ctx.Done():
    return nil, ctx.Err()
}
```

Prefer `context.WithTimeout` over `time.After` in production code — contexts are cancellable and integrate with the standard cancellation chain.

---

## The Done Channel Pattern

A `done` channel (type `chan struct{}`) signals multiple goroutines to stop by being closed:

```go
type Server struct {
    done chan struct{}
}

func NewServer() *Server {
    return &Server{done: make(chan struct{})}
}

func (s *Server) Start() {
    go s.readLoop()
    go s.writeLoop()
}

func (s *Server) Stop() {
    close(s.done)  // broadcast to all goroutines listening on s.done
}

func (s *Server) readLoop() {
    for {
        select {
        case <-s.done:
            return
        case data := <-s.incoming:
            s.process(data)
        }
    }
}
```

**Closing a channel is a broadcast.** Every goroutine blocked on `<-done` is unblocked simultaneously. This is the correct way to signal N goroutines to stop — not sending N stop values.

**Sending on a closed channel panics.** Closing on a nil channel panics. Closing an already-closed channel panics. Make sure only one goroutine closes a channel (the producer, or a dedicated closer).

---

## Pipeline Pattern

Connect stages with channels. Each stage reads from its input and writes to its output:

```go
// Stage 1: generate
func generate(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case out <- n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

// Stage 2: square
func square(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

// Usage
func main() {
    ctx := context.Background()
    nums := generate(ctx, 2, 3, 4)
    squares := square(ctx, nums)
    for v := range squares {
        fmt.Println(v)  // 4, 9, 16
    }
}
```

Key properties of correct pipeline stages:
- Each stage owns its output channel and closes it when done
- Stages accept context for cancellation
- Closing propagates downstream naturally via `range`

---

## Fan-out / Fan-in

### Fan-out: one input, multiple workers

```go
func fanOut(in <-chan Job, workerCount int) []<-chan Result {
    channels := make([]<-chan Result, workerCount)
    for i := 0; i < workerCount; i++ {
        channels[i] = worker(in)  // each worker reads from the same input
    }
    return channels
}

func worker(in <-chan Job) <-chan Result {
    out := make(chan Result)
    go func() {
        defer close(out)
        for job := range in {
            out <- process(job)
        }
    }()
    return out
}
```

### Fan-in: multiple inputs, one output (merge)

```go
func fanIn(ctx context.Context, channels ...<-chan Result) <-chan Result {
    out := make(chan Result)
    var wg sync.WaitGroup

    forward := func(ch <-chan Result) {
        defer wg.Done()
        for r := range ch {
            select {
            case out <- r:
            case <-ctx.Done():
                return
            }
        }
    }

    wg.Add(len(channels))
    for _, ch := range channels {
        go forward(ch)
    }

    // Close output when all inputs are drained
    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

---

## Channel Anti-Patterns

### Nil Channel as a Disable Switch

A receive on a nil channel blocks forever. Use this to disable a select case:

```go
func merge(a, b <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for a != nil || b != nil {
            select {
            case v, ok := <-a:
                if !ok { a = nil; continue }  // disable case when closed
                out <- v
            case v, ok := <-b:
                if !ok { b = nil; continue }  // disable case when closed
                out <- v
            }
        }
    }()
    return out
}
```

### Checking Channel Closure

Use the two-value receive to detect if a channel was closed:

```go
v, ok := <-ch
if !ok {
    // channel is closed and drained
    return
}
```

Or use `range` to automatically exit on close:

```go
for v := range ch {  // exits when ch is closed and drained
    process(v)
}
```
