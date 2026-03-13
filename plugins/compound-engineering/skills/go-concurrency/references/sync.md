# sync Package, errgroup, semaphore, and context.Context

Target: Go 1.21+

---

## sync.Mutex and sync.RWMutex

Use a `Mutex` to protect shared state that multiple goroutines read and write:

```go
type Counter struct {
    mu    sync.Mutex
    count int64
}

func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *Counter) Value() int64 {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}
```

### Mutex Rules

**Never copy a Mutex.** A Mutex must be passed by pointer, not by value. Copying a locked mutex has undefined behavior.

```go
// WRONG: copies the mutex
func process(c Counter) {  // Counter contains sync.Mutex — this is a copy!
    c.Inc()
}

// RIGHT: pass by pointer
func process(c *Counter) {
    c.Inc()
}
```

`go vet ./...` catches this: "assignment copies lock value".

**Lock the smallest scope possible.** Do not hold a lock while doing I/O, calling external services, or running expensive computation:

```go
// WRONG: holds lock during I/O
func (s *Store) FetchAndCache(id string) (Item, error) {
    s.mu.Lock()
    defer s.mu.Unlock()
    if item, ok := s.cache[id]; ok {
        return item, nil
    }
    item, err := s.db.Fetch(id)  // I/O while holding lock — deadlock risk
    if err != nil {
        return Item{}, err
    }
    s.cache[id] = item
    return item, nil
}

// RIGHT: check/update cache separately from I/O
func (s *Store) FetchAndCache(id string) (Item, error) {
    s.mu.RLock()
    item, ok := s.cache[id]
    s.mu.RUnlock()
    if ok {
        return item, nil
    }

    item, err := s.db.Fetch(id)  // I/O outside the lock
    if err != nil {
        return Item{}, err
    }

    s.mu.Lock()
    s.cache[id] = item
    s.mu.Unlock()
    return item, nil
}
```

### sync.RWMutex

Use `RWMutex` when reads are frequent and writes are rare:

```go
type Cache struct {
    mu    sync.RWMutex
    items map[string]Item
}

func (c *Cache) Get(key string) (Item, bool) {
    c.mu.RLock()            // multiple goroutines can hold RLock concurrently
    defer c.mu.RUnlock()
    item, ok := c.items[key]
    return item, ok
}

func (c *Cache) Set(key string, item Item) {
    c.mu.Lock()             // exclusive lock for writes
    defer c.mu.Unlock()
    c.items[key] = item
}
```

**RWMutex is not always faster.** It has higher overhead than Mutex for write-heavy workloads. Profile before switching.

---

## sync.WaitGroup

Use `WaitGroup` to wait for a collection of goroutines to finish:

```go
func processAll(ctx context.Context, items []Item) {
    var wg sync.WaitGroup
    for _, item := range items {
        wg.Add(1)                  // Add BEFORE launching the goroutine
        item := item               // capture loop variable (pre Go 1.22)
        go func() {
            defer wg.Done()        // Done in a defer so it runs even on panic
            process(ctx, item)
        }()
    }
    wg.Wait()
}
```

### WaitGroup Rules

- Call `Add(n)` **before** launching the goroutine — never inside the goroutine
- Always pair `Add` with exactly one `Done` (use defer)
- Do not copy a WaitGroup — pass by pointer if you need to share it

---

## sync.Once

`Once` runs a function exactly once, regardless of how many goroutines call it:

```go
type Connection struct {
    once sync.Once
    conn net.Conn
}

func (c *Connection) Get() net.Conn {
    c.once.Do(func() {
        var err error
        c.conn, err = net.Dial("tcp", "localhost:5432")
        if err != nil {
            panic(err)  // Once has no error return; panic or handle differently
        }
    })
    return c.conn
}
```

For initialization that can fail, consider `sync.OnceValue` or `sync.OnceValues` (Go 1.21+):

```go
var getConn = sync.OnceValues(func() (net.Conn, error) {
    return net.Dial("tcp", "localhost:5432")
})

// Usage:
conn, err := getConn()
```

---

## sync.Map

`sync.Map` is a concurrent-safe map optimized for two specific patterns:
1. A given key is written once and read many times (stable read set)
2. Multiple goroutines read/write disjoint sets of keys (sharded workloads)

For general-purpose concurrent maps, a `map` protected by a `sync.RWMutex` is often clearer and faster.

```go
var m sync.Map

// Store
m.Store("key", "value")

// Load
if v, ok := m.Load("key"); ok {
    fmt.Println(v.(string))
}

// LoadOrStore (atomic read-or-write)
actual, loaded := m.LoadOrStore("key", "default")

// Delete
m.Delete("key")

// Range (not snapshot-consistent — may miss concurrent writes)
m.Range(func(k, v any) bool {
    fmt.Println(k, v)
    return true  // return false to stop iteration
})
```

---

## context.Context

`context.Context` is the Go-wide standard for cancellation, deadlines, and request-scoped values.

### The Contract

- First argument to any function that may block or do I/O: `func Fetch(ctx context.Context, url string) ([]byte, error)`
- Never store `ctx` in a struct field — pass it explicitly every time
- Never pass `nil` as ctx; use `context.Background()` or `context.TODO()`

### Creating Contexts

```go
// Root contexts
ctx := context.Background()   // top-level; never cancelled
ctx := context.TODO()         // placeholder; replace with real context later

// With cancellation
ctx, cancel := context.WithCancel(parent)
defer cancel()  // always call cancel to release resources

// With deadline
ctx, cancel := context.WithDeadline(parent, time.Now().Add(5*time.Second))
defer cancel()

// With timeout (shorthand for WithDeadline)
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
defer cancel()

// With value (request-scoped data, not function parameters)
type ctxKeyType struct{}
ctx = context.WithValue(ctx, ctxKeyType{}, userID)
// Retrieve:
userID := ctx.Value(ctxKeyType{}).(int64)
```

### Checking Cancellation

```go
// In a select (for channel-based loops)
select {
case <-ctx.Done():
    return ctx.Err()  // context.Canceled or context.DeadlineExceeded
default:
    // proceed
}

// After a blocking call
if err := db.QueryContext(ctx, query); err != nil {
    if ctx.Err() != nil {
        return ctx.Err()  // context was cancelled or timed out
    }
    return fmt.Errorf("query: %w", err)
}
```

### Context Values: Use Typed Keys

Never use primitive types (string, int) as context keys — they collide across packages. Use an unexported struct type:

```go
// WRONG: any package can overwrite "userID"
ctx = context.WithValue(ctx, "userID", 42)

// RIGHT: package-specific key type prevents collisions
type contextKey struct{}
type userIDKey struct{}

ctx = context.WithValue(ctx, userIDKey{}, 42)
```

---

## errgroup (golang.org/x/sync/errgroup)

`errgroup` combines `sync.WaitGroup` with error collection. Use it when goroutines can fail and you care about the first error:

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) ([][]byte, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([][]byte, len(urls))

    for i, url := range urls {
        i, url := i, url  // capture (pre Go 1.22)
        g.Go(func() error {
            data, err := fetch(ctx, url)
            if err != nil {
                return fmt.Errorf("fetch %s: %w", url, err)
            }
            results[i] = data
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err  // first non-nil error from any goroutine
    }
    return results, nil
}
```

Key properties:
- `errgroup.WithContext` returns a derived context that is cancelled when **any** goroutine returns a non-nil error
- `g.Wait()` returns the first non-nil error; remaining goroutines must check `ctx.Done()` to exit early
- All goroutines finish before `Wait()` returns — no cancellation of goroutines in flight

### Concurrency Limit with errgroup

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(10)  // at most 10 goroutines running at once (Go 1.20+)

for _, item := range items {
    item := item
    g.Go(func() error {
        return process(ctx, item)
    })
}
return g.Wait()
```

---

## semaphore (golang.org/x/sync/semaphore)

A weighted semaphore limits concurrent access. Use it when goroutines have different "weights" (e.g., CPU-bound vs I/O-bound tasks):

```go
import "golang.org/x/sync/semaphore"

const maxWorkers = 10
sem := semaphore.NewWeighted(maxWorkers)

for _, item := range items {
    if err := sem.Acquire(ctx, 1); err != nil {
        return err  // ctx cancelled
    }
    item := item
    go func() {
        defer sem.Release(1)
        process(item)
    }()
}

// Wait for all to finish: acquire all slots (blocks until all released)
if err := sem.Acquire(ctx, maxWorkers); err != nil {
    return err
}
sem.Release(maxWorkers)
```

For simpler bounded concurrency, prefer `errgroup.SetLimit` or a buffered channel of tokens:

```go
// Channel-based semaphore (classic, no external dependency)
tokens := make(chan struct{}, 10)
for _, item := range items {
    tokens <- struct{}{}  // acquire
    go func(item Item) {
        defer func() { <-tokens }()  // release
        process(item)
    }(item)
}
```

---

## sync.Cond

`sync.Cond` implements a condition variable. Use it when goroutines need to wait for a specific state change — not just any event:

```go
type Queue struct {
    mu    sync.Mutex
    cond  *sync.Cond
    items []Item
}

func NewQueue() *Queue {
    q := &Queue{}
    q.cond = sync.NewCond(&q.mu)
    return q
}

func (q *Queue) Push(item Item) {
    q.mu.Lock()
    q.items = append(q.items, item)
    q.cond.Signal()  // wake one waiting goroutine
    q.mu.Unlock()
}

func (q *Queue) Pop() Item {
    q.mu.Lock()
    defer q.mu.Unlock()
    for len(q.items) == 0 {
        q.cond.Wait()  // atomically releases lock and sleeps; reacquires on wake
    }
    item := q.items[0]
    q.items = q.items[1:]
    return item
}
```

In most cases, a channel is cleaner than `sync.Cond`. Prefer channels unless you specifically need the broadcast semantics or multi-condition waiting that `Cond` provides.
