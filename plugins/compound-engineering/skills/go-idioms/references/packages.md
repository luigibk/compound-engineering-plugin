# Go Package Layout, internal/, cmd/, and Import Cycles

Target: Go 1.21+

---

## Standard Module Layout

A well-structured Go module separates application entry points, library code, and private packages:

```
myapp/
├── go.mod
├── go.sum
├── cmd/
│   ├── server/
│   │   └── main.go       # binary: go build ./cmd/server
│   └── migrate/
│       └── main.go       # binary: go build ./cmd/migrate
├── internal/
│   ├── auth/
│   │   ├── auth.go
│   │   └── auth_test.go
│   ├── storage/
│   │   ├── postgres.go
│   │   └── storage_test.go
│   └── config/
│       └── config.go
├── pkg/                  # (optional) packages safe to import by external callers
│   └── apierror/
│       └── apierror.go
└── README.md
```

### cmd/

- One subdirectory per binary
- Each `main.go` is thin: parse flags, build dependencies, call internal code
- `cmd/server/main.go` should be < 50 lines in most cases

```go
// cmd/server/main.go
func main() {
    cfg, err := config.Load()
    if err != nil {
        log.Fatalf("config: %v", err)
    }
    srv, err := internal.NewServer(cfg)
    if err != nil {
        log.Fatalf("server: %v", err)
    }
    if err := srv.Run(); err != nil {
        log.Fatalf("run: %v", err)
    }
}
```

### internal/

Packages under `internal/` can only be imported by code rooted at the parent of `internal/`. The Go toolchain enforces this at compile time — no annotation or convention needed.

```
myapp/internal/auth     → importable by myapp/*
myapp/internal/auth     → NOT importable by github.com/other/pkg
```

Use `internal/` for:
- Implementation packages not intended for external callers
- Domain types, database adapters, configuration structs
- Anything that would be a breaking change to make public

### pkg/

Optional. Use `pkg/` only for packages that external callers genuinely need and that you are willing to maintain as a public API. Most services do not need `pkg/` — everything goes in `internal/`.

---

## Package Granularity

### Too coarse (everything in one package)

```
myapp/
└── app/
    ├── auth.go
    ├── storage.go
    ├── handler.go
    └── model.go    // 4000 lines in one package
```

Problems: naming collisions, long compile times, hard to test in isolation.

### Too fine (one file per type)

```
myapp/internal/
├── user/user.go
├── userrepository/userrepository.go
├── userfactory/userfactory.go
```

Problems: import churn, stutter, excessive abstraction.

### Right granularity

Group by domain concept, not by layer:

```
internal/
├── user/
│   ├── user.go        # User type and methods
│   ├── store.go       # Storage interface + implementations
│   └── handler.go     # HTTP handlers for /users
├── auth/
│   ├── auth.go        # Authentication logic
│   └── token.go       # JWT encoding/decoding
```

---

## Avoiding Import Cycles

Go forbids circular imports. If package `a` imports package `b`, package `b` cannot import package `a`.

### Detecting Cycles

```bash
go build ./...
# "import cycle not allowed" error identifies the cycle
```

### Common Causes and Fixes

**Cause 1: Shared types used in two packages that import each other**

```
user imports storage
storage imports user   ← cycle
```

Fix: extract shared types to a third package that neither imports:

```
types/user.go   ← defines User struct; imports nothing
user/ imports types
storage/ imports types
```

**Cause 2: Utility function that needs a domain type**

```
util imports user   ← to use user.User
user imports util   ← to use util.Format
```

Fix: move the utility function into the `user` package, or parameterize it with an interface:

```go
// Instead of func Format(u *user.User) string in util/
// Add Format() string method to user.User
func (u *User) Format() string { ... }
```

**Cause 3: Test helper that creates a cycle**

Use external test packages (`package user_test` instead of `package user`) for test files that need to import other packages:

```go
// user/user_test.go
package user_test   // external — can import storage without creating a cycle

import (
    "myapp/internal/storage"
    "myapp/internal/user"
)
```

---

## The init() Function

`init()` runs automatically before `main()`, in dependency order. Minimize its use.

**Acceptable uses:**
- Registering drivers or codecs: `database/sql` driver registration, `image` format registration
- One-time computation of package-level variables that cannot be const

```go
// Acceptable: registering a database driver
func init() {
    sql.Register("postgres", &Driver{})
}
```

**Avoid:**
- Any I/O, network calls, or file reads in `init()`
- Mutation of global state that other packages depend on
- Complex logic that could panic

If you need to initialize something with potential errors, use a `New()` function or explicit `Initialize()` that returns an error.

---

## Package-Level Variables

Minimize mutable global state. Prefer dependency injection over singletons:

```go
// BAD: global mutable state
var db *sql.DB

func init() {
    var err error
    db, err = sql.Open("postgres", os.Getenv("DATABASE_URL"))
    // ...
}

// GOOD: explicit dependency injection
type Server struct {
    db *sql.DB
}

func NewServer(db *sql.DB) *Server {
    return &Server{db: db}
}
```

Package-level `var` is acceptable for:
- Sentinel errors: `var ErrNotFound = errors.New("not found")`
- Compile-time interface checks: `var _ io.Reader = (*MyReader)(nil)`
- Unexported constants that cannot be `const`: compiled regexps, etc.

```go
// Acceptable: pre-compiled regex
var reEmail = regexp.MustCompile(`^[^@]+@[^@]+\.[^@]+$`)
```

---

## File Organization Within a Package

No strict rule, but these conventions are widely followed:

- `doc.go` — package documentation comment, single `package` line
- `{packagename}.go` — primary exported types and their methods
- `{feature}.go` — grouping by feature for large packages
- `{type}_test.go` — tests in the same package (white-box) or `package foo_test` (black-box)
- `errors.go` — sentinel errors and custom error types for the package
- `options.go` — functional options pattern if the package uses it

Keep files under ~400 lines. When a file grows beyond that, split it by concept rather than by line count.
