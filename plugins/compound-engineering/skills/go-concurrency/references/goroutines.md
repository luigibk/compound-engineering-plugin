# Goroutines: Lifecycle Management, Leak Prevention, Worker Pools

Target: Go 1.21+

---

## Goroutine Ownership

Every goroutine must have an **owner** — the code that launches it is responsible for ensuring it terminates. Document ownership explicitly:

```go
// Owner: Server.Run() — goroutine lives for the duration of the server
func (s *Server) Run(ctx context.Context) error {
    go s.collectMetrics(ctx)  // exits when ctx is cancelled
    return s.listenAndServe(ctx)
}
```

If a goroutine is launched deep in a library without a clear owner, it is almost always a leak waiting to happen. Goroutines launched by libraries must accept a context or return a stop function.

---

## Three Goroutine Exit Patterns

### Pattern 1: Context Cancellation (preferred for I/O and long-running work)

```go
func (s *Subscriber) Subscribe(ctx context.Context, topic string) {
    go func() {
        for {
            select {
            case <-ctx.Done():
                return  // clean exit
            case msg := <-s.messages:
                s.handler(msg)
            }
        }
    }()
}
```

### Pattern 2: Channel Close (preferred for pipelines and producer/consumer)

```go
func process(jobs <-chan Job) {
    go func() {
        for job := range jobs {  // exits when jobs is closed
            job.Execute()
        }
    }()
}

// Caller signals completion by closing the channel
close(jobs)
```

### Pattern 3: Done Channel (explicit signal)

```go
type Worker struct {
    done chan struct{}
}

func (w *Worker) Start() {
    go func() {
        for {
            select {
            case <-w.done:
                return
            default:
                w.doWork()
            }
        }
    }()
}

func (w *Worker) Stop() {
    close(w.done)  // broadcast to all goroutines listening on w.done
}
```

---

## Detecting Goroutine Leaks

### Using goleak in Tests

The `goleak` package (go.uber.org/goleak) detects goroutines that are still running after a test ends:

```go
func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}

func TestSubscriber(t *testing.T) {
    defer goleak.VerifyNone(t)  // fails if goroutines are leaked

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    sub := NewSubscriber()
    sub.Start(ctx)
    // test logic...
    // cancel() is deferred — goroutine exits before test ends
}
```

### Manual Inspection

```bash
# Build with race detector — catches races but not leaks
go test -race ./...

# Profile goroutines at runtime
import _ "net/http/pprof"
# Then: curl http://localhost:6060/debug/pprof/goroutine?debug=1
```

---

## Worker Pool Pattern

Use a worker pool to bound concurrent goroutines for CPU-bound or rate-limited work:

```go
func processItems(ctx context.Context, items []Item, workerCount int) error {
    jobs := make(chan Item, len(items))
    errs := make(chan error, workerCount)

    // Launch fixed number of workers
    var wg sync.WaitGroup
    for i := 0; i < workerCount; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for item := range jobs {
                if err := process(ctx, item); err != nil {
                    errs <- err
                    return
                }
            }
        }()
    }

    // Feed work
    for _, item := range items {
        jobs <- item
    }
    close(jobs)  // signal workers to exit when drained

    // Wait for all workers to finish
    wg.Wait()
    close(errs)

    // Collect errors
    for err := range errs {
        if err != nil {
            return err
        }
    }
    return nil
}
```

### Worker Pool with errgroup (simpler)

```go
import "golang.org/x/sync/errgroup"
import "golang.org/x/sync/semaphore"

func processItems(ctx context.Context, items []Item, maxConcurrent int64) error {
    sem := semaphore.NewWeighted(maxConcurrent)
    g, ctx := errgroup.WithContext(ctx)

    for _, item := range items {
        item := item  // capture loop variable (required before Go 1.22)
        if err := sem.Acquire(ctx, 1); err != nil {
            return err
        }
        g.Go(func() error {
            defer sem.Release(1)
            return process(ctx, item)
        })
    }

    return g.Wait()
}
```

---

## Common Goroutine Bugs

### Bug 1: Goroutine Launched in a Loop Without Capturing Loop Variable

```go
// WRONG (before Go 1.22): all goroutines share the same 'i' variable
for i := 0; i < 10; i++ {
    go func() {
        fmt.Println(i)  // prints "10" ten times, not 0-9
    }()
}

// RIGHT (works in all Go versions): capture the variable
for i := 0; i < 10; i++ {
    i := i  // new variable per iteration
    go func() {
        fmt.Println(i)
    }()
}
// Note: Go 1.22+ fixes loop variable capture automatically
```

### Bug 2: WaitGroup.Add Inside the Goroutine

```go
// WRONG: if main goroutine reaches wg.Wait() before the goroutine calls wg.Add, wait returns immediately
var wg sync.WaitGroup
go func() {
    wg.Add(1)     // too late!
    defer wg.Done()
    doWork()
}()
wg.Wait()  // may return before doWork() runs

// RIGHT: Add before launching
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    doWork()
}()
wg.Wait()
```

### Bug 3: Goroutine Launched Without a Stop Mechanism

```go
// WRONG: this goroutine runs forever; no way to stop it
func startPoller() {
    go func() {
        for {
            pollExternalService()
            time.Sleep(5 * time.Second)
        }
    }()
}

// RIGHT: accept context for cancellation
func startPoller(ctx context.Context) {
    go func() {
        ticker := time.NewTicker(5 * time.Second)
        defer ticker.Stop()
        for {
            select {
            case <-ctx.Done():
                return
            case <-ticker.C:
                pollExternalService()
            }
        }
    }()
}
```

### Bug 4: Goroutine Leak on Error Path

```go
// WRONG: if the caller returns early due to error, the goroutine blocks forever on resultCh
func compute(input int) (int, error) {
    resultCh := make(chan int)  // unbuffered!
    go func() {
        resultCh <- expensiveCompute(input)  // blocks forever if nobody reads
    }()

    if err := validate(input); err != nil {
        return 0, err  // goroutine is now leaked
    }
    return <-resultCh, nil
}

// RIGHT: use a buffered channel so the goroutine can always send
func compute(input int) (int, error) {
    resultCh := make(chan int, 1)  // buffered: goroutine never blocks
    go func() {
        resultCh <- expensiveCompute(input)
    }()

    if err := validate(input); err != nil {
        return 0, err  // goroutine exits normally after sending
    }
    return <-resultCh, nil
}
```

---

## Goroutines vs Threads

| Property | OS Thread | Goroutine |
|----------|-----------|-----------|
| Stack size | 1-8 MB (fixed) | 2-8 KB (grows dynamically) |
| Context switch | Kernel, ~1μs | User-space, ~100ns |
| Practical count | Hundreds to low thousands | Millions |
| Communication | Shared memory + locks | Channels (preferred) or shared memory |

Goroutines are cheap enough to launch one per request in a server — but they are not free. A million idle goroutines each holding a 2 KB stack use 2 GB of memory. Use worker pools when processing large batches.
