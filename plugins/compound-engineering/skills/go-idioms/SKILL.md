---
name: go-idioms
description: Idiomatic Go patterns, naming conventions, interface design, error handling, and tooling. Use when writing Go code, designing package APIs, reviewing Go patterns, or when Effective Go conventions are needed.
---

<objective>
Apply idiomatic Go conventions drawn from Effective Go, the Go team's style guide, and patterns from well-maintained open-source Go projects. Target Go 1.21+.
</objective>

<essential_principles>
## Core Philosophy

"Clear is better than clever." — Rob Pike

**Go is deliberately opinionated:**
- Simplicity over abstraction: flat package structures beat deep hierarchies
- Composition over inheritance: embed types, define small interfaces at the consumer
- Explicit error handling: errors are values, not exceptions
- Naming as documentation: short names for narrow scope, longer names for wider scope
- Zero values should be useful: design types so the zero value works without initialization

**What Go deliberately omits:**
- Exceptions (use multiple return values with error)
- Method overloading (impossible in Go)
- Circular imports (enforced by the compiler at compile time)
- Implicit interface satisfaction declarations (a type implements an interface by having the methods)
- Inheritance (use embedding and interface composition)
</essential_principles>

<intake>
What are you working on?

1. **Naming** — packages, types, functions, variables, constants, acronyms
2. **Interfaces** — design, placement, composition, the consumer-side rule
3. **Errors** — wrapping, sentinel errors, custom error types, errors.Is/As
4. **Packages** — layout, internal/, cmd/, avoiding import cycles
5. **Embedding** — struct embedding, promoted methods, interface embedding
6. **Tooling** — build tags, go:generate, ldflags, cross-compilation
7. **Code Review** — review Go code against idiomatic conventions
8. **General Guidance** — philosophy and conventions for a Go task

**Specify a number or describe your task.**
</intake>

<routing>

| Response | Reference to Read |
|----------|-------------------|
| 1, naming | [patterns.md](./references/patterns.md) |
| 2, interface | [patterns.md](./references/patterns.md) |
| 3, error | [errors.md](./references/errors.md) |
| 4, package, layout, cycle | [packages.md](./references/packages.md) |
| 5, embed | [patterns.md](./references/patterns.md) |
| 6, tool, build, generate, cross, ldflags | [tooling.md](./references/tooling.md) |
| 7, review | Read all references, then review code against them |
| 8, general task | Read relevant references based on context |

**After reading relevant references, apply the patterns to the user's code.**
</routing>

<quick_reference>
## Naming at a Glance

**Packages:** lowercase, single word, no underscores, no plurals.
- `http`, `json`, `user` — not `utils`, `helpers`, `common`, `lib`
- Package name is part of the API: `user.User` is stuttering; the type should be `user.Profile` or the package `users`

**Exported identifiers:** MixedCaps. Acronyms stay fully upper-cased.
- `UserID`, `HTTPClient`, `ParseURL` — not `UserId`, `HttpClient`, `ParseUrl`

**Unexported identifiers:** lowerCamelCase.
- `maxRetries`, `parseToken`, `errNotFound`

**Interfaces:** one-method interfaces use `-er` suffix.
- `Reader`, `Writer`, `Closer`, `Stringer`
- Multi-method: descriptive noun — `http.Handler`, `io.ReadWriter`

**Receivers:** one or two lowercase letters, consistent within a type.
- `func (u *User) Save()` — not `func (user *User) Save()` or `func (this *User) Save()`

**Error variables:** exported sentinel errors are `ErrX` variables, never `ErrXError`.
- `var ErrNotFound = errors.New("not found")` — not `ErrNotFoundError`

## Interface Design (Consumer-Side Rule)

Define interfaces where they are **used**, not where they are **implemented**:

```go
// WRONG: package user owns the interface — forces every caller to import user
package user
type Repository interface {
    Get(ctx context.Context, id int64) (*User, error)
}

// RIGHT: each consumer defines exactly the methods it needs
package handler
type userGetter interface {
    Get(ctx context.Context, id int64) (*user.User, error)
}
func NewHandler(users userGetter) *Handler { ... }
```

Keep interfaces small. Prefer `io.Reader` (one method) over a six-method interface. If you can test a function by passing a small interface, you have the right boundary.

## Error Wrapping (Quick)

Always add context when returning errors:

```go
// WRONG: caller cannot tell where the error came from
if err != nil {
    return err
}

// RIGHT: wrap with context using %w so errors.Is / errors.As work
if err != nil {
    return fmt.Errorf("get user %d: %w", id, err)
}
```

Inspect errors with `errors.Is` (identity) and `errors.As` (type):

```go
if errors.Is(err, ErrNotFound) { ... }

var ve *ValidationError
if errors.As(err, &ve) { ... }
```
</quick_reference>

<reference_index>
## Domain Knowledge

All detailed patterns in `references/`:

| File | Topics |
|------|--------|
| [patterns.md](./references/patterns.md) | Naming rules, interface design, struct embedding, zero values, defer, receivers |
| [packages.md](./references/packages.md) | Package layout, internal/, cmd/, avoiding import cycles, init() rules |
| [errors.md](./references/errors.md) | Error wrapping, sentinel errors, custom error types, errors.Is/As, panic vs error |
| [tooling.md](./references/tooling.md) | Build tags, go:generate, ldflags version injection, cross-compilation |
</reference_index>

<success_criteria>
Code follows idiomatic Go when:
- Package names are short, lowercase, single words with no stutter
- Interfaces are defined at the consumer with one or two methods
- Errors are always wrapped with `fmt.Errorf("context: %w", err)`, never silently dropped
- All exported names use MixedCaps with acronyms fully capitalized (URL, ID, HTTP)
- Struct embedding is used for composition; no type hierarchies
- The zero value of every type is safe to use without explicit initialization
- No circular imports; no unexported identifiers in package-level `init()` that cross packages
- `errors.Is` / `errors.As` are used for error inspection, not string comparison
</success_criteria>
