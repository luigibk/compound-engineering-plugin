---
name: go-modules
description: Go module management including go.mod anatomy, replace directives, workspace mode, vendoring, and dependency upgrade workflow. Use when managing dependencies, debugging module errors, working with replace directives, or setting up multi-module workspaces.
---

# Go Modules

Manage Go module dependencies with confidence. Covers `go.mod`, `go.sum`, the module graph, workspace mode, and vendoring. Target Go 1.21+.

---

## go.mod Anatomy

Every Go module starts with a `go.mod` file at the root:

```
module github.com/myorg/myapp

go 1.21

require (
    github.com/lib/pq v1.10.9
    golang.org/x/sync v0.6.0
)

require (
    github.com/lib/pq v1.10.9 // indirect
)
```

### Directives

| Directive | Purpose |
|-----------|---------|
| `module` | The module's import path — must match how it is imported by callers |
| `go` | Minimum Go toolchain version; also controls language semantics |
| `require` | Direct and indirect dependencies with exact versions |
| `replace` | Override a module's source (local fork, path override) |
| `exclude` | Exclude a specific version from the build graph |
| `retract` | Mark versions as retracted (published in error) |

### Direct vs Indirect Dependencies

```
require (
    github.com/lib/pq v1.10.9                    // direct: imported in your code
    golang.org/x/text v0.14.0 // indirect         // indirect: required by a dependency
)
```

After running `go mod tidy`, the `// indirect` comment is managed automatically. Do not edit it by hand.

---

## replace Directives

`replace` overrides where Go fetches a module. Two legitimate uses:

### 1. Local Fork During Development

```
replace github.com/upstream/lib => ../local-fork/lib
```

Points to a local directory. **Must be removed before merging to main.** A replace pointing to a local path cannot be resolved by other developers or CI.

**Check for accidental replace directives:**

```bash
grep -n "^replace" go.mod
# Any replace pointing to a relative path (../) is a red flag in production
```

### 2. Forked Dependency on a Remote VCS

```
replace github.com/upstream/lib => github.com/myorg/lib-fork v1.2.3-fork
```

This is acceptable in production but documents a maintenance burden. Note why the fork exists in a comment above the directive.

### Anti-Patterns

```
# WRONG: local path left in production go.mod
replace github.com/upstream/lib => ../lib

# WRONG: replace with no version (ties you to a pseudo-version)
replace github.com/upstream/lib => github.com/myorg/lib-fork
```

---

## Module Graph Commands

```bash
# List all direct and indirect dependencies with versions
go list -m all

# Show why a dependency is in the build graph
go mod why golang.org/x/sync
go mod why -m golang.org/x/text  # module-level (includes indirect paths)

# Print the full dependency graph
go mod graph

# Visualize with modgraph (install: go install golang.org/x/mod/cmd/modgraph@latest)
go mod graph | modgraph

# Check for available updates
go list -m -u all

# Update a specific module to its latest minor/patch release
go get github.com/lib/pq@latest

# Update all direct dependencies
go get -u ./...

# Remove unused dependencies and add missing ones
go mod tidy
```

---

## go mod tidy

`go mod tidy` is the canonical way to synchronize `go.mod` and `go.sum` with actual imports:

```bash
go mod tidy
```

What it does:
1. Adds missing `require` entries for packages imported in `.go` files
2. Removes `require` entries for packages no longer imported
3. Updates `go.sum` to match
4. Adds `// indirect` comments for transitive dependencies

**Run `go mod tidy` before every commit** that changes imports. CI should fail if `go.mod` is not tidy:

```bash
# CI check: verify go.mod is already tidy
go mod tidy
git diff --exit-code go.mod go.sum
```

---

## go.sum: The Verification File

`go.sum` records the cryptographic hash of every downloaded module version. Never edit it by hand.

```
github.com/lib/pq v1.10.9 h1:YX...
github.com/lib/pq v1.10.9/go.mod h1:Ab...
```

**Always commit `go.sum`** alongside `go.mod`. It is not a lock file — it is a security verification mechanism. Deleting it forces Go to re-verify every dependency on the next build.

---

## Upgrading Dependencies

### Check for available updates

```bash
go list -m -u all
# Output: module current [available]
# github.com/lib/pq v1.10.7 [v1.10.9]
```

### Upgrade a specific module

```bash
go get github.com/lib/pq@v1.10.9     # exact version
go get github.com/lib/pq@latest      # latest release
go get github.com/lib/pq@main        # latest commit on main (not recommended for stability)
```

### Upgrade all direct dependencies (minor/patch only)

```bash
go get -u ./...
go mod tidy
```

### Major version upgrades

Major versions (`v2`, `v3`) are separate modules with different import paths:

```go
// v1 import
import "github.com/go-redis/redis"

// v2+ import — note the /v2 suffix
import "github.com/go-redis/redis/v2"
```

```bash
go get github.com/go-redis/redis/v9@latest
# Update all import paths in .go files to use /v9
```

---

## Workspace Mode (go.work)

Go workspaces (Go 1.18+) allow working on multiple modules in the same directory without using `replace` directives. This solves the "local replace hack" pattern.

### Creating a Workspace

```bash
# In a directory containing multiple module roots:
go work init ./myapp ./mylib
```

This creates `go.work`:

```
go 1.21

use (
    ./myapp
    ./mylib
)
```

When `go.work` exists, Go uses the local versions of listed modules instead of the registry.

### Workspace Rules

- **Do not commit `go.work` to a shared repository** (add to `.gitignore`). It is a developer-local convenience.
- Use `go.work` instead of `replace` for multi-module development in a monorepo or adjacent repositories.
- CI builds should set `GOWORK=off` to ensure they use registered versions:

```bash
GOWORK=off go build ./...
GOWORK=off go test ./...
```

### Workspace Commands

```bash
go work init       # create go.work
go work use ./lib  # add a module to go.work
go work sync       # sync dependencies between go.work modules
```

---

## Vendoring

Vendoring copies all dependencies into a `vendor/` directory so builds can run without network access:

```bash
go mod vendor        # create/update vendor/
go build -mod=vendor ./...  # build using vendor/
go test -mod=vendor ./...
```

### When to Vendor

**Use vendoring when:**
- Builds must be hermetic (airgapped environments, regulated industries)
- CI has no reliable network access
- The module is a library and you want reproducible builds without `GONOSUMDB`

**Skip vendoring when:**
- The module proxy and `go.sum` verification are sufficient (most cases)
- The vendor directory size is prohibitive (large dependency trees)

### Vendoring Rules

- Commit the entire `vendor/` directory
- Run `go mod vendor` and commit the result whenever dependencies change
- CI should verify the vendor directory is up to date:

```bash
go mod vendor
git diff --exit-code vendor/
```

---

## Vulnerabilities: govulncheck

`govulncheck` reports known vulnerabilities in the module's dependency graph, filtered to only vulnerabilities that affect code paths actually called:

```bash
# Install
go install golang.org/x/vuln/cmd/govulncheck@latest

# Run
govulncheck ./...

# Check specific module (even without calling code)
govulncheck -mode=module ./...
```

Integrate into CI to catch vulnerabilities on every PR. The GitHub Actions `govulncheck-action` runs it automatically.

---

## Common Module Errors

### "no required module provides package..."

A package is imported in `.go` source but not in `go.mod`:

```bash
go mod tidy  # adds the missing require entry
```

### "go: inconsistent vendoring"

The `vendor/` directory is out of sync with `go.mod`:

```bash
go mod vendor  # regenerate vendor/
```

### "verifying module: checksum mismatch"

`go.sum` does not match the downloaded module. Causes: corrupted download, module was deleted and re-uploaded, `GONOSUMCHECK` misconfiguration:

```bash
go clean -modcache  # clear module cache
go mod download     # re-download all modules
```

### "ambiguous import: found package X in multiple modules"

Two `replace` directives or two modules in `go.work` both provide the same package. Remove the duplication.
