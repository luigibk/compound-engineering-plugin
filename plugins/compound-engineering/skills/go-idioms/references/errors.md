# Go Error Handling: Wrapping, Sentinels, Custom Types, errors.Is/As

Target: Go 1.21+

---

## Core Principle: Errors Are Values

Go errors are plain values implementing the `error` interface:

```go
type error interface {
    Error() string
}
```

Errors are returned, not thrown. Every call site decides what to do with an error. This explicitness is intentional — the compiler ensures you cannot silently ignore a returned error without assigning it to `_`.

---

## Always Wrap Errors with Context

Add context at every layer boundary so that a stack of wrapped errors reads like a narrative from outermost to innermost cause:

```go
// WRONG: loses all context — the caller has no idea where this came from
func getUser(id int64) (*User, error) {
    row := db.QueryRow("SELECT * FROM users WHERE id = $1", id)
    var u User
    if err := row.Scan(&u.Name, &u.Email); err != nil {
        return nil, err  // caller sees "sql: no rows in result set" — from where?
    }
    return &u, nil
}

// RIGHT: caller sees "handler get user 42: query user 42: sql: no rows in result set"
func getUser(ctx context.Context, id int64) (*User, error) {
    row := db.QueryRowContext(ctx, "SELECT * FROM users WHERE id = $1", id)
    var u User
    if err := row.Scan(&u.Name, &u.Email); err != nil {
        return nil, fmt.Errorf("query user %d: %w", id, err)
    }
    return &u, nil
}

func (h *Handler) handleGetUser(w http.ResponseWriter, r *http.Request) {
    id := parseID(r)
    u, err := getUser(r.Context(), id)
    if err != nil {
        // error chain: "handler get user 42: query user 42: sql: no rows in result set"
        log.Printf("handler get user %d: %v", id, err)
        http.Error(w, "not found", http.StatusNotFound)
        return
    }
    // ...
}
```

### Wrapping Rules

- Use `fmt.Errorf("context: %w", err)` to wrap and preserve the original error for `errors.Is`/`errors.As`
- Do NOT use `%v` for wrapping — it creates a string, breaking the error chain
- Convention: context string ends without a period, uses lowercase, colon-separates levels

```go
fmt.Errorf("open config file: %w", err)      // good
fmt.Errorf("Open config file: %w", err)      // bad — title case
fmt.Errorf("failed to open config file: %w", err)  // OK but verbose; "open config file:" is enough
```

---

## Sentinel Errors

Sentinel errors are package-level `var` values used as known, comparable error identities:

```go
// In package storage
var (
    ErrNotFound    = errors.New("not found")
    ErrConflict    = errors.New("conflict")
    ErrPermission  = errors.New("permission denied")
)
```

### When to Use Sentinels

- When callers need to branch on a specific well-known condition
- When the error has no additional context beyond its identity
- When the error is part of the package's public API contract

### Checking Sentinels: errors.Is

`errors.Is` unwraps the error chain and checks identity:

```go
u, err := store.GetUser(ctx, id)
if errors.Is(err, storage.ErrNotFound) {
    http.Error(w, "user not found", http.StatusNotFound)
    return
}
if err != nil {
    http.Error(w, "internal error", http.StatusInternalServerError)
    return
}
```

**Do NOT use string comparison or `==` for wrapped errors:**

```go
// WRONG: fails if the error was wrapped
if err == storage.ErrNotFound { ... }
if err.Error() == "not found" { ... }

// RIGHT: works through any number of wrapping layers
if errors.Is(err, storage.ErrNotFound) { ... }
```

---

## Custom Error Types

Use a custom error type when the error needs to carry structured data:

```go
// Custom error type — carries fields the caller can inspect
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation: %s: %s", e.Field, e.Message)
}

// Returning it
func validateAge(age int) error {
    if age < 0 {
        return &ValidationError{Field: "age", Message: "must be non-negative"}
    }
    return nil
}
```

### Checking Custom Types: errors.As

`errors.As` unwraps the chain looking for the first error that matches the target type:

```go
err := validateAge(-1)
if err != nil {
    var ve *ValidationError
    if errors.As(err, &ve) {
        // ve.Field == "age", ve.Message == "must be non-negative"
        http.Error(w, fmt.Sprintf("invalid field: %s", ve.Field), http.StatusBadRequest)
        return
    }
    http.Error(w, "internal error", http.StatusInternalServerError)
    return
}
```

### Implementing errors.Is on Custom Types

For custom types that wrap another error, implement `Unwrap()` to participate in the chain:

```go
type DBError struct {
    Op  string
    Err error  // the underlying error
}

func (e *DBError) Error() string { return fmt.Sprintf("db %s: %v", e.Op, e.Err) }
func (e *DBError) Unwrap() error { return e.Err }  // enables errors.Is/As unwrapping
```

For custom types that wrap multiple errors (e.g., an aggregate), implement `Unwrap() []error`:

```go
type MultiError struct {
    Errs []error
}

func (e *MultiError) Error() string { ... }
func (e *MultiError) Unwrap() []error { return e.Errs }  // Go 1.20+
```

---

## panic vs error

Use `panic` only for truly unrecoverable states — programmer errors, not runtime errors:

```go
// Acceptable: programming error, configuration impossible to correct at runtime
func mustCompileRegexp(pattern string) *regexp.Regexp {
    re, err := regexp.Compile(pattern)
    if err != nil {
        panic(fmt.Sprintf("invalid regexp %q: %v", pattern, err))
    }
    return re
}

var reEmail = mustCompileRegexp(`^[^@]+@[^@]+\.[^@]+$`)
```

**Never use panic for expected error conditions** (not found, validation failure, network error). Always return an `error` value.

Libraries especially must not panic on runtime errors — a panic in a library propagates to the caller's goroutine.

---

## Error Handling Patterns

### Early Return (Guard Clause)

Prefer returning early on error over deeply nested if-else:

```go
// WRONG: pyramid of doom
func process(r io.Reader) error {
    if data, err := io.ReadAll(r); err == nil {
        if parsed, err := parse(data); err == nil {
            if err = save(parsed); err == nil {
                return nil
            } else {
                return err
            }
        } else {
            return err
        }
    } else {
        return err
    }
}

// RIGHT: flat early returns
func process(r io.Reader) error {
    data, err := io.ReadAll(r)
    if err != nil {
        return fmt.Errorf("read: %w", err)
    }
    parsed, err := parse(data)
    if err != nil {
        return fmt.Errorf("parse: %w", err)
    }
    if err := save(parsed); err != nil {
        return fmt.Errorf("save: %w", err)
    }
    return nil
}
```

### Logging Errors Once

Log an error at the top of the call stack, not at every layer. Wrapping provides context; logging at every layer creates duplicate log lines:

```go
// WRONG: logs "db error" at storage layer AND "failed to get user" at handler layer
func (s *Store) GetUser(id int64) (*User, error) {
    u, err := s.db.Get(id)
    if err != nil {
        log.Printf("db error: %v", err)  // log here
        return nil, fmt.Errorf("get user: %w", err)
    }
    return u, nil
}

func (h *Handler) GetUser(w http.ResponseWriter, r *http.Request) {
    u, err := h.store.GetUser(id)
    if err != nil {
        log.Printf("failed to get user: %v", err)  // AND log again
        // ...
    }
}

// RIGHT: wrap without logging; log once at the boundary
func (s *Store) GetUser(id int64) (*User, error) {
    u, err := s.db.Get(id)
    if err != nil {
        return nil, fmt.Errorf("get user %d: %w", id, err)  // wrap only
    }
    return u, nil
}

func (h *Handler) GetUser(w http.ResponseWriter, r *http.Request) {
    u, err := h.store.GetUser(id)
    if err != nil {
        log.Printf("handler: %v", err)  // log once, at the boundary
        http.Error(w, "not found", http.StatusNotFound)
    }
}
```

---

## Checking for Go 1.13+ Error Features

All error wrapping features require Go 1.13+. In Go 1.20+, `errors.Join` and `Unwrap() []error` (multi-error chains) are available. Target Go 1.21 to use all features.

```go
// Go 1.20+: joining multiple errors
err := errors.Join(err1, err2)
// errors.Is(err, err1) == true
// errors.Is(err, err2) == true
```
