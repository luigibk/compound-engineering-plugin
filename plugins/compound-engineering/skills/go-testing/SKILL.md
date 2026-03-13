---
name: go-testing
description: Table-driven tests, subtests, test helpers, benchmarks, and fuzz testing for Go. Use when writing or reviewing Go tests, generating test cases for existing functions, setting up benchmarks, or adding fuzz targets. Target Go 1.21+.
---

# Go Testing

Write idiomatic Go tests using the standard `testing` package. Go testing is intentionally minimal — no assertion library is required, and the patterns are highly consistent across the ecosystem.

## Table-Driven Tests

The canonical Go test structure. Collect all test cases in a slice, then run them in a loop with `t.Run`:

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name    string
        a, b    int
        want    int
        wantErr bool
    }{
        {name: "positive numbers", a: 2, b: 3, want: 5},
        {name: "negative plus positive", a: -1, b: 4, want: 3},
        {name: "zeros", a: 0, b: 0, want: 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.want {
                t.Errorf("Add(%d, %d) = %d, want %d", tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

### Anatomy of a Test Case Struct

| Field | Purpose |
|-------|---------|
| `name string` | Subtest name; appears in `go test -v` output and on failure |
| `input` fields | Input values for the function under test |
| `want` / `wantOut` | Expected return value |
| `wantErr bool` | Whether an error is expected (use when function returns error) |
| `wantErrIs error` | Specific sentinel error to check with `errors.Is` |

### Running a Subtest

```bash
go test -run TestAdd/zeros ./...
go test -run TestAdd -v ./...
```

### Table-Driven Tests with Errors

```go
func TestParseID(t *testing.T) {
    tests := []struct {
        name      string
        input     string
        want      int64
        wantErr   bool
        wantErrIs error
    }{
        {name: "valid id", input: "42", want: 42},
        {name: "negative", input: "-1", wantErr: true},
        {name: "not a number", input: "abc", wantErr: true, wantErrIs: ErrInvalidID},
        {name: "empty string", input: "", wantErr: true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseID(tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("ParseID(%q) error = %v, wantErr %v", tt.input, err, tt.wantErr)
                return
            }
            if tt.wantErrIs != nil && !errors.Is(err, tt.wantErrIs) {
                t.Errorf("ParseID(%q) error = %v, want errors.Is(%v)", tt.input, err, tt.wantErrIs)
            }
            if err == nil && got != tt.want {
                t.Errorf("ParseID(%q) = %d, want %d", tt.input, got, tt.want)
            }
        })
    }
}
```

---

## Subtests and t.Run

`t.Run` creates a named subtest. Each subtest runs in its own goroutine if `t.Parallel()` is called inside:

```go
func TestHandler(t *testing.T) {
    t.Run("GET returns 200", func(t *testing.T) {
        t.Parallel()  // run this subtest concurrently with others
        // ...
    })
    t.Run("POST returns 201", func(t *testing.T) {
        t.Parallel()
        // ...
    })
}
```

Call `t.Parallel()` immediately inside the subtest function, before any test logic. The parent test waits for all subtests to complete before returning.

**Loop variable capture with parallel subtests (pre Go 1.22):**

```go
for _, tt := range tests {
    tt := tt  // capture! Required before Go 1.22
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        // use tt safely
    })
}
```

---

## Test Helpers with t.Helper()

Mark assertion helpers with `t.Helper()` so that failure output points to the call site, not inside the helper:

```go
func assertNoError(t *testing.T, err error) {
    t.Helper()  // makes failures report the caller's line, not this function
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
}

func assertEqual[T comparable](t *testing.T, got, want T) {
    t.Helper()
    if got != want {
        t.Errorf("got %v, want %v", got, want)
    }
}

// Usage
func TestProcess(t *testing.T) {
    result, err := Process("input")
    assertNoError(t, err)       // failure reports this line
    assertEqual(t, result, 42)  // failure reports this line
}
```

### testing.TB for Shared Helpers

Accept `testing.TB` (the interface satisfied by `*testing.T`, `*testing.B`, and `*testing.F`) in helpers shared between tests and benchmarks:

```go
func newTestDB(tb testing.TB) *sql.DB {
    tb.Helper()
    db, err := sql.Open("sqlite3", ":memory:")
    if err != nil {
        tb.Fatalf("open test db: %v", err)
    }
    tb.Cleanup(func() { db.Close() })
    return db
}
```

---

## t.Cleanup

Register cleanup functions with `t.Cleanup` instead of `defer` when building test fixtures:

```go
func setupServer(t *testing.T) *httptest.Server {
    t.Helper()
    srv := httptest.NewServer(newHandler())
    t.Cleanup(srv.Close)  // runs after the test, even on failure
    return srv
}
```

`t.Cleanup` functions run in LIFO order, same as defer. Prefer it over defer for test fixtures because it works correctly with `t.Parallel` subtests.

---

## testify (Optional)

The standard library's `testing` package is sufficient for most tests. `testify/assert` and `testify/require` are popular third-party additions that provide readable assertions:

```go
import (
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestUser(t *testing.T) {
    u, err := NewUser("alice@example.com")
    require.NoError(t, err)          // stops the test immediately on failure (like t.Fatal)
    assert.Equal(t, "alice", u.Name) // records failure but continues (like t.Error)
    assert.True(t, u.Active)
}
```

| Package | Behavior on failure |
|---------|---------------------|
| `assert` | Records failure and continues (`t.Errorf`) |
| `require` | Stops the test immediately (`t.Fatalf`) |

Use `require` for preconditions (if the next assertion would panic on nil), `assert` for independent checks.

**Rule of thumb:** Prefer stdlib for simple projects and libraries. Use testify when deeply nested struct comparisons or message formatting become tedious.

---

## Benchmarks

Benchmarks live in `_test.go` files and use `func BenchmarkX(b *testing.B)`:

```go
func BenchmarkConcat(b *testing.B) {
    b.ReportAllocs()  // report memory allocations per operation

    for i := 0; i < b.N; i++ {
        _ = strings.Join([]string{"hello", "world", "go"}, " ")
    }
}
```

### Benchmark Rules

- Call `b.ReportAllocs()` at the start — always measure allocations
- Reset the timer if setup work happens before the timed loop:

```go
func BenchmarkProcess(b *testing.B) {
    data := generateTestData(1000)  // setup
    b.ResetTimer()                  // start timing here
    b.ReportAllocs()

    for i := 0; i < b.N; i++ {
        _ = Process(data)
    }
}
```

- Prevent the compiler from optimizing away the result by assigning to a package-level `var`:

```go
var result int

func BenchmarkAdd(b *testing.B) {
    var r int
    for i := 0; i < b.N; i++ {
        r = Add(i, i+1)
    }
    result = r  // prevents dead-code elimination
}
```

### Running Benchmarks

```bash
go test -bench=. -benchmem ./...          # all benchmarks, with memory stats
go test -bench=BenchmarkProcess -count=5  # run 5 times for statistical stability
go test -bench=. -benchtime=10s ./...     # run for at least 10 seconds
```

Output format:
```
BenchmarkConcat-10    5000000    234 ns/op    48 B/op    1 allocs/op
```

---

## Fuzz Testing (Go 1.18+)

Fuzz tests find inputs that crash or cause unexpected behavior. They are written alongside unit tests in `_test.go` files:

```go
func FuzzParseInput(f *testing.F) {
    // Seed corpus: known interesting inputs
    f.Add("hello")
    f.Add("")
    f.Add("123")
    f.Add("special\x00chars")

    f.Fuzz(func(t *testing.T, input string) {
        // Must not panic for any input
        result, err := ParseInput(input)
        if err != nil {
            return  // errors are acceptable; panics are not
        }
        // Invariant: round-trip must be stable
        if got := result.String(); got != input {
            t.Errorf("round-trip failed: ParseInput(%q).String() = %q", input, got)
        }
    })
}
```

### Running Fuzz Tests

```bash
# Run only the seed corpus (fast, suitable for CI)
go test -run FuzzParseInput ./...

# Run the fuzzer actively (generates new inputs; run locally)
go test -fuzz=FuzzParseInput -fuzztime=60s ./...
```

Fuzz-found failures are saved to `testdata/fuzz/FuzzX/` and are automatically replayed by `go test -run`. Commit these files so CI catches regressions.

### What to Fuzz

Fuzz testing is most valuable for:
- Parsers (JSON, CSV, custom formats)
- Deserializers and decoders
- Functions that process untrusted input
- Encoding/decoding round-trips
- Compression and decompression

---

## Test Organization

```
mypackage/
├── user.go
├── user_test.go         # package user — white-box tests (can access unexported fields)
└── user_integration_test.go  # package user_test — black-box tests (external perspective)
```

Use `package foo_test` (external test package) for tests that exercise the public API and may import other packages. Use `package foo` (same package) when you need to test unexported functions or access internal state.

```bash
# Run tests with -v to see subtest names
go test -v ./...

# Run tests matching a pattern
go test -run TestUser ./...

# Run with race detector (mandatory for concurrent code)
go test -race ./...

# Generate coverage report
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```
