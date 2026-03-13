# Go Patterns: Naming, Interfaces, Embedding, Zero Values, Defer

Target: Go 1.21+

---

## Naming Rules

### Package Names

A package name is part of every identifier exported from it. Choose names that make the combination read naturally:

```
// Package "user" — exported type is user.User (stutter — bad)
// Package "users" — plural (bad)
// Package "auth" — user.Authenticate reads naturally (good)
// Package "profile" — profile.Profile still stutters (bad); profile.Summary is fine
```

Rules:
- Lowercase, single word, no underscores, no MixedCaps
- Avoid generic names: `util`, `common`, `lib`, `misc`, `helpers` — these tell the reader nothing
- If you must have a utility package, name it after what it does: `httpclient`, `timeutil`
- Test packages use `_test` suffix: `package user_test` for black-box tests

### Identifier Names

**Exported (visible outside the package):** MixedCaps, acronyms fully capitalized.

```go
// Correct
type UserID int64
type HTTPClient struct{}
func ParseURL(raw string) (*url.URL, error)
const MaxRetries = 3

// Wrong
type UserId int64       // acronym not fully capitalized
type HttpClient struct{} // acronym not fully capitalized
func ParseUrl(raw string) (*url.URL, error)
```

**Unexported (package-private):** lowerCamelCase, acronyms still capitalized internally when part of a compound name.

```go
var errNotFound = errors.New("not found")  // lowercase e
var maxHTTPRetries = 5                      // HTTP stays capitalized
func parseJWT(token string) (*Claims, error) // JWT stays capitalized
```

**Local variables:** short names are idiomatic inside functions. Single letters for loop indexes, receivers, and tight scopes:

```go
for i, v := range items { ... }

func (u *User) Validate() error { ... }  // u, not user or this or self

// Longer names for wider scope or when type is non-obvious
func processPayment(ctx context.Context, amount int64, currency string) error { ... }
```

**Boolean names:** avoid `IsX`, `HasX` for unexported fields; prefer plain adjectives:

```go
type Server struct {
    running bool   // not isRunning
    healthy bool   // not isHealthy
}
```

For exported methods, `IsX` is conventional when it disambiguates:

```go
func (s *Server) IsRunning() bool  // fine for exported method
```

---

## Interface Design

### The Consumer-Side Rule

Interfaces belong with the code that **uses** them, not the code that implements them.

```go
// WRONG: producer defines the interface in its own package
// package storage
type Storage interface {
    Get(ctx context.Context, key string) ([]byte, error)
    Set(ctx context.Context, key string, val []byte) error
    Delete(ctx context.Context, key string) error
}

// RIGHT: each consumer defines only what it needs
// package handler
type keyValueGetter interface {
    Get(ctx context.Context, key string) ([]byte, error)
}

// package reaper
type keyDeleter interface {
    Delete(ctx context.Context, key string) error
}
```

Benefits:
- Decoupling: `handler` does not import `storage`
- Testability: inject a small fake struct in tests, not a mock of a 10-method interface
- Flexibility: any type with `Get(ctx, key)` satisfies `keyValueGetter` without a declaration

### Interface Size

One or two methods. The Go standard library is the model:

```
io.Reader       — Read(p []byte) (n int, err error)
io.Writer       — Write(p []byte) (n int, err error)
io.Closer       — Close() error
io.ReadCloser   — Read + Close (composed)
http.Handler    — ServeHTTP(ResponseWriter, *Request)
fmt.Stringer    — String() string
```

If you find yourself writing a six-method interface, ask: do all callers need all six methods? Usually the answer is no — split it.

### Interface Naming Conventions

```
io.Reader       → verb + er suffix (single-method)
io.ReadWriter   → compound of two Reader + Writer (composed)
http.Handler    → noun (multi-method, describes a role)
sort.Interface  → reserved for the sort package convention
```

### Compile-Time Interface Check

Add a blank assignment to verify a type implements an interface at compile time:

```go
var _ io.Reader = (*MyReader)(nil)
// Fails to compile if *MyReader doesn't implement io.Reader
```

Place this immediately below the type definition.

### Return Concrete Types, Accept Interfaces

```go
// Function parameters: accept interface (flexible)
func Process(r io.Reader) error { ... }

// Return values: return concrete type (informative)
func NewFile(name string) *os.File { ... }  // not io.ReadWriteCloser
```

Exception: return an interface when the concrete type is unexported or the abstraction is intentional (e.g., `error`, `http.Handler`).

---

## Struct Embedding

Embedding promotes methods from the embedded type to the outer type without wrapping:

```go
type Logger struct {
    *log.Logger          // promotes Logger's methods to outer type
    prefix string
}

// Callers can call l.Printf() directly
l := &Logger{Logger: log.New(os.Stdout, "", 0), prefix: "app"}
l.Printf("started")
```

### When to Embed

- To add behavior to a type without reimplementing every method (decorator pattern)
- To compose interface implementations from smaller parts

```go
type ReadWriteBuffer struct {
    io.Reader   // provides Read
    io.Writer   // provides Write
    buf         []byte
}
```

### When NOT to Embed

- When you want to hide the embedded type's methods from the outer API
- When embedding would create ambiguous method sets

If you embed to hide, use a named field instead:

```go
type Server struct {
    logger *log.Logger   // not embedded — callers cannot call s.Printf()
}
```

---

## Zero Values

Design types so the zero value is immediately usable without initialization:

```go
// Good: zero value works
var b bytes.Buffer
b.WriteString("hello")  // no New() needed

var wg sync.WaitGroup
wg.Add(1)  // no init needed

// Good: exported type with safe zero value
type Counter struct {
    mu    sync.Mutex
    count int64
}
func (c *Counter) Inc() { c.mu.Lock(); c.count++; c.mu.Unlock() }
// Counter{} is immediately usable — no constructor required
```

When a zero value is not safe, require construction but make the error obvious:

```go
// BAD: zero value panics
type Client struct {
    conn net.Conn  // nil conn will panic on use
}

// GOOD: document the constructor, fail fast
func NewClient(addr string) (*Client, error) {
    conn, err := net.Dial("tcp", addr)
    if err != nil {
        return nil, fmt.Errorf("new client: %w", err)
    }
    return &Client{conn: conn}, nil
}
```

---

## Defer

Use `defer` for cleanup that must happen regardless of how a function exits:

```go
func copyFile(dst, src string) error {
    in, err := os.Open(src)
    if err != nil {
        return fmt.Errorf("open src: %w", err)
    }
    defer in.Close()

    out, err := os.Create(dst)
    if err != nil {
        return fmt.Errorf("create dst: %w", err)
    }
    defer out.Close()

    _, err = io.Copy(out, in)
    return err
}
```

### Defer Pitfalls

**Defer in a loop:** defers accumulate until the function returns, not the loop iteration. Extract to a helper:

```go
// WRONG: defers pile up if processFiles is long-running
for _, f := range files {
    fd, _ := os.Open(f)
    defer fd.Close()  // only runs when processFiles returns
    // ...
}

// RIGHT: extract to a function
for _, f := range files {
    if err := processFile(f); err != nil {
        return err
    }
}
func processFile(path string) error {
    fd, err := os.Open(path)
    if err != nil { return err }
    defer fd.Close()  // runs when processFile returns
    // ...
}
```

**Named return values with defer:** mutation of named return values in a deferred function works:

```go
func double(n int) (result int, err error) {
    defer func() {
        if err == nil {
            result *= 2
        }
    }()
    result, err = compute(n)
    return
}
```

Use this pattern for wrapping errors or committing transactions — not for general logic.

---

## Receiver Rules

- Use a pointer receiver when the method needs to modify the receiver, or the struct is large
- Use a value receiver when the method only reads, and the type is small or a value type (like `time.Time`)
- Be consistent: if any method uses a pointer receiver, all methods should use pointer receivers

```go
// Consistent: all pointer receivers
func (s *Server) Start() error  { ... }
func (s *Server) Stop() error   { ... }
func (s *Server) Addr() string  { return s.addr }  // reads only, but stays pointer for consistency

// Value receiver for small immutable type
func (p Point) Distance(q Point) float64 { ... }
```

Never use a pointer receiver on a type that is used as a map key or stored by value in a slice.
