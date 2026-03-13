# Add Go Language Support to compound-engineering

> **Scope:** 6 new agents + 4 new skills + updates to 5 existing files
> **Version target:** 2.36.0 (MINOR — new agents and skills)
> **Current counts:** 29 agents · 22 commands · 19 skills
> **After counts:** 35 agents · 22 commands · 23 skills

---

## 1. Inventory of Existing Language-Specific Content

### Existing language reviewers

All three follow the **`kieran-{language}-reviewer`** naming pattern and live in `plugins/compound-engineering/agents/review/`:

| File | Language | Persona |
|------|----------|---------|
| `kieran-rails-reviewer.md` | Ruby/Rails | Kieran as senior Rails dev |
| `kieran-python-reviewer.md` | Python | Kieran as senior Python dev |
| `kieran-typescript-reviewer.md` | TypeScript | Kieran as senior TS dev |

A fourth reviewer uses a different persona:

| File | Language | Persona |
|------|----------|---------|
| `dhh-rails-reviewer.md` | Ruby/Rails | DHH — opinionated 37signals style |

There is also a language-specific workflow agent:

| File | Language |
|------|----------|
| `workflow/lint.md` | Ruby + ERB (runs standardrb, erblint, brakeman) |

### Existing language-specific skills

| Skill dir | Language |
|-----------|----------|
| `skills/dhh-rails-style/` | Ruby/Rails — comprehensive reference with `references/` subdir |
| `skills/andrew-kane-gem-writer/` | Ruby gems — medium-weight with `references/` subdir |
| `skills/dspy-ruby/` | Ruby + LLM — heavy reference (700+ lines, `references/` + `assets/`) |

**No Go-specific content exists anywhere in the plugin.**

### Observed naming patterns

- Language reviewers: `kieran-{language}-reviewer` (the primary series)
- Opinionated persona reviewers: `{persona}-{language}-reviewer` (DHH variant; only one exists)
- Workflow agents: action-oriented nouns (`lint`, `bug-reproduction-validator`)
- Research agents: researcher/analyst nouns (`best-practices-researcher`, `repo-research-analyst`)
- Skills: kebab-case matching the directory name, description must state "what + when"
- All agent files sit directly in their category subdir; no nested `go/` subdirectory for agents
- All skill dirs sit directly under `skills/`; no nested `go/` subdirectory for skills

### The `setup` skill: current language detection

`plugins/compound-engineering/skills/setup/SKILL.md` auto-detects stack with this shell script:

```bash
test -f Gemfile && test -f config/routes.rb && echo "rails" || \
test -f Gemfile && echo "ruby" || \
test -f tsconfig.json && echo "typescript" || \
test -f package.json && echo "javascript" || \
test -f pyproject.toml && echo "python" || \
test -f requirements.txt && echo "python" || \
echo "general"
```

Go is not detected. The "Stack" customization question lists Rails, Python, TypeScript — Go is absent. The auto-configure defaults table has no Go row.

---

## 2. Proposed Go Agents

### 2.1 `kieran-go-reviewer` — INCLUDE

**File:** `plugins/compound-engineering/agents/review/kieran-go-reviewer.md`
**Slug:** `kieran-go-reviewer`
**Model:** `inherit`

**Purpose:** Primary Go code review agent. The direct analogue of `kieran-rails-reviewer`, `kieran-python-reviewer`, and `kieran-typescript-reviewer` — Kieran as a super-senior Go engineer with an opinionated, high-quality bar. Covers the full surface of idiomatic Go: package layout, naming, error handling patterns (`fmt.Errorf("%w")`, `errors.Is`/`errors.As`), interface design (small, implicit, behavior-driven), struct embedding, defer patterns, nil safety, and the "no unnecessary abstractions" philosophy.

**Go pain points addressed:**
- Over-engineered interfaces (defining interfaces at the producer instead of the consumer)
- Swallowed errors (`if err != nil { return }` without wrapping context)
- Leaking implementation details through exported types
- God packages (everything in one `main` or `util` package)
- Non-idiomatic naming (`getUser` instead of `User`, `isValid` instead of `Valid`)
- Missing table-driven tests for pure functions
- Receiver naming inconsistency

**Tools the agent uses:** Read, Glob, Bash (for `go vet`, `go build ./...`)

**Analogues in repo:** Direct parallel of all three `kieran-*-reviewer` agents. Same structure: persona intro, numbered principles with FAIL/PASS examples, reviewing philosophy.

---

### 2.2 `go-concurrency-reviewer` — INCLUDE

**File:** `plugins/compound-engineering/agents/review/go-concurrency-reviewer.md`
**Slug:** `go-concurrency-reviewer`
**Model:** `inherit`

**Purpose:** Specialized review agent for Go's concurrency primitives. Go's goroutines, channels, sync primitives, and context propagation have enough footguns to warrant a dedicated reviewer — more so than Python threading or JS async/await. This agent catches the class of bugs that compile cleanly and pass unit tests but cause production incidents.

**Go pain points addressed:**
- Goroutine leaks (goroutines launched without a completion signal or cancel path)
- Channel directionality not enforced (`chan T` instead of `chan<- T` / `<-chan T`)
- Mutex being copied by value (passed as struct field, not pointer)
- `sync.WaitGroup` misuse (calling `Add` inside a goroutine)
- Missing `context.Context` propagation — blocking operations not respect cancellation
- Race conditions on shared maps without sync
- Unbounded goroutine spawning in loops without a worker pool
- `select` with no `default` causing unexpected blocking

**Tools the agent uses:** Read, Bash (`go build -race ./...`, `grep -r "go func"`)

**Analogues in repo:** `julik-frontend-races-reviewer` is the closest parallel — a specialized reviewer for a language-specific class of race condition. No direct Go equivalent exists.

**Relationship to `kieran-go-reviewer`:** Concurrency review is deep enough to run as a parallel agent alongside the primary reviewer; `kieran-go-reviewer` notes concurrency smells but does not audit them systematically.

---

### 2.3 `go-module-analyzer` — INCLUDE

**File:** `plugins/compound-engineering/agents/research/go-module-analyzer.md`
**Slug:** `go-module-analyzer`
**Model:** `inherit`

**Purpose:** Research agent that reads `go.mod` and `go.sum`, reports the module graph, flags outdated direct dependencies, identifies indirect dependency bloat, and checks for known vulnerability advisories via `govulncheck`. This is a read-and-report agent, not a code style reviewer — it belongs in the **research** category.

**Go pain points addressed:**
- `replace` directives left in production (common dev leftover that blocks upgrades)
- Dependencies pinned to pre-release or pseudo-versions for stable libraries
- Indirect dependencies that are out of date and carry CVEs
- Module graph conflicts — multiple versions of the same package
- Workspace mode (`go.work`) misuse that leaks local paths into the module

**Tools the agent uses:** Read (`go.mod`, `go.sum`), Bash (`go list -m -json all`, `go mod why`, `govulncheck ./...`)

**Analogues in repo:** `schema-drift-detector` is the closest model — a focused research/analysis agent that reads specific artifact files and reports findings. No other dependency-graph agent exists.

**Exclude: `go-migration-helper`** — dep→modules migration is a historical one-time operation that the vast majority of Go projects have already completed. The ROI does not justify the maintenance cost of an agent that almost no user will need.

---

### 2.4 `go-test-writer` — INCLUDE

**File:** `plugins/compound-engineering/agents/workflow/go-test-writer.md`
**Slug:** `go-test-writer`
**Model:** `inherit`

**Purpose:** Workflow agent that writes idiomatic Go tests for existing functions. Go testing is highly opinionated: table-driven tests in `_test.go` files, subtests via `t.Run`, no global test state, `testify` is optional not required, `testing.T.Helper()` for assertion helpers, and benchmark functions follow `func BenchmarkX(b *testing.B)`. This agent produces tests, not reviews — it belongs in the **workflow** category.

**Go pain points addressed:**
- Boilerplate friction for writing table-driven tests (most engineers know the pattern; the cost is writing it)
- Tests that only test the happy path and miss zero values, nil inputs, empty slices
- Using external test helpers when `testing.TB` suffices
- Benchmarks that don't use `b.ReportAllocs()` or reset the timer correctly
- Fuzz test setup for string-input functions

**Tools the agent uses:** Read, Write, Glob

**Analogues in repo:** No direct equivalent. `kieran-rails-reviewer` calls out missing tests but does not write them. This fills the gap between "reviewer flags missing test" and "developer writes the test".

**Exclude: `go-migration-helper`** — already excluded above.

---

### 2.5 `go-linter-advisor` — INCLUDE

**File:** `plugins/compound-engineering/agents/workflow/go-linter-advisor.md`
**Slug:** `go-linter-advisor`
**Model:** `haiku`

**Purpose:** Lightweight workflow agent (haiku model, like the `lint` agent) that runs `golangci-lint` for the project, interprets the output, groups findings by linter and severity, and auto-fixes what it can via `golangci-lint run --fix`. Analogous to the existing `lint` agent, which runs standardrb/erblint for Ruby.

**Go pain points addressed:**
- `golangci-lint` output is verbose and hard to triage — 50+ linters fire, many at once
- Default config enables too few linters; recommended configs vary by project type
- `go vet` catches some issues but not structural ones (shadow, wrapcheck, exhaustive)
- Engineers skip linting locally because the DX is poor

**Tools the agent uses:** Bash (`golangci-lint run`, `go vet ./...`, `staticcheck ./...`)

**Analogues in repo:** Direct parallel to `workflow/lint.md` which runs standardrb + erblint + brakeman.

**Relationship to `go-interface-designer`:** Interface design guidance is better delivered as part of `kieran-go-reviewer` (which reviews produced code) and the `go-idioms` skill (which teaches the consumer-interface pattern). A standalone `go-interface-designer` agent would be too narrow and would duplicate `kieran-go-reviewer`. **Exclude `go-interface-designer`.**

---

### 2.6 `go-performance-profiler` — INCLUDE

**File:** `plugins/compound-engineering/agents/review/go-performance-profiler.md`
**Slug:** `go-performance-profiler`
**Model:** `inherit`

**Purpose:** Go-specific performance review agent. The generic `performance-oracle` handles algorithmic complexity and N+1 query patterns but does not know Go's memory model, escape analysis, allocation patterns, or pprof tooling. This agent audits code for heap escape (`&x` leaking to heap), unnecessary allocations in hot paths, `strings.Builder` vs `+` concatenation, slice growth patterns, and guides pprof CPU and memory profiling.

**Go pain points addressed:**
- Variables escaping to heap unnecessarily (identified by `go build -gcflags='-m'`)
- Slice pre-allocation omitted in known-size loops
- `fmt.Sprintf` used where `strconv` or direct concatenation is faster
- Memory profile interpretation — confusing inuse vs alloc
- Premature use of `sync.Pool` without benchmarks proving it helps
- Interface boxing causing allocations in hot paths

**Tools the agent uses:** Bash (`go test -bench`, `go build -gcflags='-m'`, `go test -memprofile`), Read

**Analogues in repo:** Extends `performance-oracle` for Go specifics. `performance-oracle` remains the general-purpose agent; `go-performance-profiler` is invoked when the project is detected as Go (similar to how `kieran-rails-reviewer` supplements the generic review for Rails projects).

**Exclude: standalone `go-error-handler`** — Error wrapping patterns (fmt.Errorf, errors.Is/As) are important but are a section within `kieran-go-reviewer`, not a standalone agent. A dedicated agent would be too narrow (would mostly read 3-5 lines per call site) and would overlap heavily with the primary reviewer.

---

## 3. Proposed Go Skills

### 3.1 `go-idioms` — INCLUDE

**Directory:** `plugins/compound-engineering/skills/go-idioms/`
**Files:**
```
skills/go-idioms/
├── SKILL.md           # Main entry point (intake router + quick reference)
└── references/
    ├── patterns.md    # Effective Go patterns: naming, interfaces, embedding, errors
    ├── packages.md    # Package layout, import cycles, internal/, cmd/ structure
    ├── errors.md      # Error wrapping, sentinel errors, custom error types
    └── tooling.md     # Build tags, ldflags, go:generate, cross-compilation
```

**What this skill encodes:**
- Naming conventions: `MixedCaps` for exported, `mixedCaps` for unexported, no underscores
- Consumer-side interface design (define interfaces where you use them, not where you implement them)
- Error wrapping with `%w` and the `errors.Is`/`errors.As` inspection chain
- Struct embedding for composition without inheritance
- Zero values as useful defaults
- Package naming: short, lowercase, no stutter (`http.Handler` not `http.HTTPHandler`)
- `internal/` for unexported packages, `cmd/` for executables
- Build tags and `go:generate` directives

**Which agents or commands invoke it:**
- `kieran-go-reviewer` — loads `patterns.md` and `errors.md` for review context
- `best-practices-researcher` — already scans SKILL.md files for relevant skills; will surface `go-idioms` automatically when researching Go topics
- `setup` skill — referenced in Go auto-configure description

**Rationale:** `dhh-rails-style` is the template: a comprehensive reference skill that agents read before reviewing/writing code. `go-idioms` fills the equivalent role for Go. The references subdirectory allows deep content without an unnavigably long SKILL.md.

---

### 3.2 `go-testing` — INCLUDE

**Directory:** `plugins/compound-engineering/skills/go-testing/`
**Files:**
```
skills/go-testing/
├── SKILL.md           # Table-driven tests, subtests, helpers pattern
└── references/
    ├── table-driven.md  # Complete table-driven test anatomy with examples
    ├── benchmarks.md    # Benchmark setup, ReportAllocs, reset timer, profiling
    └── fuzz.md          # Go 1.18+ fuzz testing setup and corpus management
```

**What this skill encodes:**
- Table-driven test anatomy: `name`, `input`, `want`, `wantErr` fields
- `t.Run` for subtests with parallel execution (`t.Parallel()`)
- Test helper functions using `t.Helper()` and `testing.TB`
- Testify vs stdlib trade-offs (testify adds readable assertions, stdlib has zero deps)
- Benchmark pitfalls: compiler optimization defeating benchmarks, `b.ResetTimer()` usage
- Fuzz test corpus format and how to incorporate seed corpus

**Which agents or commands invoke it:**
- `go-test-writer` — reads this skill before generating tests
- `kieran-go-reviewer` — references this when flagging missing tests

---

### 3.3 `go-concurrency` — INCLUDE

**Directory:** `plugins/compound-engineering/skills/go-concurrency/`
**Files:**
```
skills/go-concurrency/
├── SKILL.md           # Quick reference: goroutines, channels, sync, context
└── references/
    ├── goroutines.md  # Lifecycle management, leak prevention, worker pools
    ├── channels.md    # Directionality, select patterns, fan-out/fan-in
    └── sync.md        # Mutex, RWMutex, WaitGroup, Once, errgroup
```

**What this skill encodes:**
- Goroutine lifecycle: always have an exit condition; document the owner
- Channel directionality: send-only `chan<-`, receive-only `<-chan` in function signatures
- The `context.Context` contract: first argument, never stored in structs
- Worker pool pattern for bounded concurrency
- `sync.WaitGroup` correct usage (Add before goroutine launch, not inside)
- `errgroup.Group` from `golang.org/x/sync` for goroutines with error propagation
- Data race detection: `go test -race ./...` is mandatory for concurrent code

**Which agents or commands invoke it:**
- `go-concurrency-reviewer` — reads this as its reference document
- `go-test-writer` — references goroutine testing patterns when generating tests for concurrent code

---

### 3.4 `go-modules` — INCLUDE (lean)

**Directory:** `plugins/compound-engineering/skills/go-modules/`
**Files:**
```
skills/go-modules/
└── SKILL.md           # Single file; no references/ needed at this scope
```

**What this skill encodes (SKILL.md only — no references/ subdir):**
- `go.mod` anatomy: module path, go directive, require, replace, exclude
- `replace` directives: legitimate uses (local forks) vs dangerous uses (left in production)
- Module graph commands: `go mod graph`, `go mod why`, `go mod tidy`
- Workspace mode (`go.work`): when to use it, how to not commit it
- Vendoring: `go mod vendor` and when it's appropriate (hermetic builds, offline CI)
- Upgrade workflow: `go get -u ./...`, `go mod tidy`

**Rationale for lean format (no references/):** Module management is a narrower topic than Go idioms or concurrency. A single well-structured SKILL.md (~150 lines) covers the ground. The `dspy-ruby` model is appropriate for `go-idioms`; the `andrew-kane-gem-writer` / `git-worktree` model is appropriate here.

**Which agents or commands invoke it:**
- `go-module-analyzer` — reads this before running analysis
- `kieran-go-reviewer` — references when reviewing go.mod changes in a PR

**Exclude: `go-toolchain` as a separate skill** — Build tags, ldflags, cross-compilation, and go:generate are important but narrow enough to fit in `go-idioms/references/tooling.md`. A separate `go-toolchain` skill would have very limited invocation surface and would be mostly duplicating content. Fold into `go-idioms`.

---

## 4. Changes Required to Existing Files

### 4.1 `plugins/compound-engineering/.claude-plugin/plugin.json`

**Field:** `version`
- Current: `"2.35.2"`
- New: `"2.36.0"` (MINOR bump — new agents and skills)

**Field:** `description`
- Current: `"AI-powered development tools. 29 agents, 22 commands, 19 skills, 1 MCP server for code review, research, design, and workflow automation."`
- New: `"AI-powered development tools. 35 agents, 22 commands, 23 skills, 1 MCP server for code review, research, design, and workflow automation."`

**Field:** `keywords` — add `"go"` and `"golang"` to the existing array.

**No other fields change.** The plugin.json does not currently contain an agents list or components object, so no structural changes are needed there.

---

### 4.2 `.claude-plugin/marketplace.json`

**Path:** `.claude-plugin/marketplace.json` (repo root)

**Field:** `plugins[0].version`
- Current: `"2.35.2"`
- New: `"2.36.0"`

**Field:** `plugins[0].description`
- Current: `"AI-powered development tools that get smarter with every use. Make each unit of engineering work easier than the last. Includes 29 specialized agents, 22 commands, and 19 skills."`
- New: `"AI-powered development tools that get smarter with every use. Make each unit of engineering work easier than the last. Includes 35 specialized agents, 22 commands, and 23 skills."`

**Field:** `plugins[0].tags` — add `"go"` and `"golang"`.

---

### 4.3 `plugins/compound-engineering/README.md`

**Section: Component table** (lines ~7-12)
Update counts:
```markdown
| Agents | 35 |
| Skills | 23 |
```

**Section: Review agents table** — currently "Review (15)". New count: Review (18).
Add three new rows:
```markdown
| `kieran-go-reviewer`      | Go code review with strict idiomatic conventions |
| `go-concurrency-reviewer` | Goroutine leaks, channel misuse, race conditions |
| `go-performance-profiler` | pprof guidance, escape analysis, allocation hotspots |
```

**Section: Research agents table** — currently "Research (5)". New count: Research (6).
Add:
```markdown
| `go-module-analyzer` | Analyze go.mod/go.sum, dependency health, vulnerabilities |
```

**Section: Workflow agents table** — currently "Workflow (5)". New count: Workflow (7).
Add:
```markdown
| `go-test-writer`   | Write table-driven tests following Go conventions |
| `go-linter-advisor` | Run and interpret golangci-lint for Go projects |
```

**Section: Skills** — add a new "Go Development" subsection:
```markdown
### Go Development

| Skill | Description |
|-------|-------------|
| `go-idioms`      | Idiomatic Go patterns, naming, interfaces, errors, tooling |
| `go-testing`     | Table-driven tests, benchmarks, fuzz testing |
| `go-concurrency` | Goroutines, channels, sync primitives, context propagation |
| `go-modules`     | Module graph, replace directives, workspace mode, vendoring |
```

---

### 4.4 `plugins/compound-engineering/skills/setup/SKILL.md`

This file drives the `/setup` flow and requires three targeted changes:

**Change 1 — Detection script** (Step 2: Detect and Ask, bash block):
Add Go detection before the `echo "general"` fallback:
```bash
test -f go.mod && echo "go" || \
```

**Change 2 — Auto-configure defaults table** (Step 4: Build Agent List and Write File):
Add a Go row:
```
- **Go:** `[kieran-go-reviewer, go-concurrency-reviewer, code-simplicity-reviewer, security-sentinel, go-performance-profiler]`
```

**Change 3 — Stack question options** (Step 3a: Customize → Stack):
Add a Go option to the `AskUserQuestion` block:
```
- label: "Go"
  description: "Go — adds idiomatic Go reviewer and concurrency reviewer"
```

And in Step 4's stack-specific agent mapping:
```
- Go → `kieran-go-reviewer, go-concurrency-reviewer`
```

---

### 4.5 `plugins/compound-engineering/CHANGELOG.md`

Add a new entry at the top for version 2.36.0 documenting all additions. Follow the Keep a Changelog format already used. Example structure:

```markdown
## [2.36.0] - YYYY-MM-DD

### Added

- **`kieran-go-reviewer` agent** — Go code review with an extremely high quality bar...
- **`go-concurrency-reviewer` agent** — ...
- **`go-module-analyzer` agent** — ...
- **`go-test-writer` agent** — ...
- **`go-linter-advisor` agent** — ...
- **`go-performance-profiler` agent** — ...
- **`go-idioms` skill** — ...
- **`go-testing` skill** — ...
- **`go-concurrency` skill** — ...
- **`go-modules` skill** — ...

### Changed

- **`setup` skill** — Added Go stack detection (`go.mod`), auto-configure defaults, and stack option.
- **`plugin.json`** — Updated version to 2.36.0 and component counts.
- **`marketplace.json`** — Updated version and description counts.
```

---

### 4.6 Workflow commands — minimal change needed

`commands/workflows/review.md` itself does NOT need editing. It delegates agent selection entirely to the `setup` skill's `compound-engineering.local.md` output. By updating `setup/SKILL.md` to include a Go stack option, the review workflow automatically gains Go-specific agents for projects that run setup.

`commands/workflows/work.md` and other workflow commands similarly do not need changes — they read from the same settings file.

No commands are added or removed.

---

## 5. Implementation Order

The following sequence respects all inter-dependencies: skills come first because agents reference them; the `setup` skill update comes last among plugin files because it lists agents by name.

```
Step 1  — Create skills/go-idioms/SKILL.md + references/
Step 2  — Create skills/go-concurrency/SKILL.md + references/
Step 3  — Create skills/go-testing/SKILL.md + references/
Step 4  — Create skills/go-modules/SKILL.md  (single file, no references/)
Step 5  — Create agents/review/kieran-go-reviewer.md
Step 6  — Create agents/review/go-concurrency-reviewer.md
Step 7  — Create agents/review/go-performance-profiler.md
Step 8  — Create agents/research/go-module-analyzer.md
Step 9  — Create agents/workflow/go-test-writer.md
Step 10 — Create agents/workflow/go-linter-advisor.md
Step 11 — Update skills/setup/SKILL.md (references agent names from steps 5-10)
Step 12 — Update plugin.json (version + description counts)
Step 13 — Update marketplace.json (version + description + tags)
Step 14 — Update README.md (component counts + tables)
Step 15 — Update CHANGELOG.md (document all additions)
Step 16 — Run /release-docs to regenerate documentation site
Step 17 — Validate JSON: cat .claude-plugin/marketplace.json | jq . && cat plugins/compound-engineering/.claude-plugin/plugin.json | jq .
Step 18 — Verify counts match: grep -o "35 agents" + ls agents/review/ etc.
```

**Rationale for this order:**
- Skills first — `go-concurrency-reviewer` should be written after `go-concurrency/SKILL.md` exists because the reviewer body cross-references the skill
- Agents before setup — `setup/SKILL.md` names agents by slug; writing agents first means you can copy exact slugs
- Metadata files after content — plugin.json/marketplace.json counts must reflect actual file counts
- `/release-docs` last — regenerates HTML from final state

---

## 6. Acceptance Criteria

### For each Go agent file

- [ ] YAML frontmatter contains `name`, `description`, and `model`
- [ ] `description` follows the format: "Verb phrase. Use when [trigger condition]." (matching existing reviewers)
- [ ] At least 2 `<examples>` blocks with `Context`, `user`, `assistant`, and `<commentary>`
- [ ] The commentary in each example uses the correct agent slug
- [ ] Agent body uses numbered sections with `## N. SECTION NAME` headings
- [ ] FAIL/PASS examples use emoji pattern (🔴 FAIL / ✅ PASS) for any stylistic rules
- [ ] The agent body is self-contained — a reviewer can verify behavior without reading other files
- [ ] The agent does not duplicate content already in an existing cross-language agent (e.g., do not re-explain Big-O notation covered by `performance-oracle`)

### For `kieran-go-reviewer` specifically

- [ ] Covers: naming, package structure, error wrapping, interface design, zero values, defer usage
- [ ] Cites `go vet` and `go build ./...` as tools it will run
- [ ] Explicitly excludes concurrency (defers to `go-concurrency-reviewer`)
- [ ] Core philosophy section echoes the "duplication > complexity" principle used in the other Kieran reviewers

### For `go-concurrency-reviewer` specifically

- [ ] Covers goroutine leaks, WaitGroup misuse, channel directionality, mutex copy, context propagation
- [ ] References `go build -race ./...` as a mandatory tool
- [ ] Cross-references `go-concurrency` skill by name

### For `go-module-analyzer` specifically

- [ ] Uses Bash tool to run `go list -m -json all`, `go mod tidy --check`, `go mod why`
- [ ] Checks for `replace` directives and flags ones pointing to local paths
- [ ] Reports findings in a structured table (module, current, latest, CVEs)

### For `go-test-writer` specifically

- [ ] Generates table-driven tests, not sequential `TestX` functions
- [ ] Uses `t.Run` for subtests
- [ ] Generates `wantErr bool` or `wantErr error` column for error cases
- [ ] Does not force testify — generates stdlib-compatible tests by default, notes testify as option

### For `go-linter-advisor` specifically

- [ ] `model: haiku` (matching `lint` agent — lightweight execution)
- [ ] Runs `golangci-lint run` and auto-fixes where `--fix` is safe
- [ ] Groups output by linter name, not by file
- [ ] Does not modify business logic — only applies linter auto-fixes

### For `go-performance-profiler` specifically

- [ ] Uses `go build -gcflags='-m' ./...` to identify heap escapes
- [ ] Does not overlap with `performance-oracle`'s algorithmic complexity coverage
- [ ] Distinguishes between premature optimization warnings and genuine hotspot findings

### For each Go skill SKILL.md

- [ ] YAML frontmatter: `name` matches directory name (lowercase-with-hyphens), `description` states what + when
- [ ] All files in `references/` linked as markdown links (no backtick-only references)
- [ ] Writing style is imperative/objective — no "you should"
- [ ] Quick reference section with code examples in Go (not pseudocode)
- [ ] `go-idioms` and `go-concurrency` have populated `references/` subdirectories
- [ ] `go-modules` and `go-testing` are self-contained in SKILL.md (no references/ needed)

### For metadata and config updates

- [ ] `plugin.json` version is `2.36.0`
- [ ] `plugin.json` description contains "35 agents, 22 commands, 23 skills"
- [ ] `marketplace.json` version and description match `plugin.json` exactly
- [ ] `README.md` component table shows agents=35, skills=23
- [ ] `README.md` review section heading shows "(18)"
- [ ] `README.md` research section heading shows "(6)"
- [ ] `README.md` workflow section heading shows "(7)"
- [ ] `setup/SKILL.md` Go detection finds `go.mod` and produces correct auto-configure list
- [ ] `CHANGELOG.md` entry dated and formatted per Keep a Changelog spec
- [ ] `jq .` passes on both JSON files with no errors
- [ ] `/release-docs` completes without validation errors

---

## 7. Open Questions

### Q1: Persona for `kieran-go-reviewer` — use Kieran, or use a Go community figure?

The existing reviewers are all "Kieran as {language} expert." There is no Rob Pike or Russ Cox persona analogous to DHH for Rails. Options:

- **Option A:** Keep "Kieran as Go expert" — consistent with the series, no persona research required
- **Option B:** Use Rob Pike persona — more opinionated and memorable; requires studying his public code/talks for authenticity; adds maintenance risk if the persona diverges from community consensus

**Recommendation:** Option A for the initial implementation. A Rob Pike variant can always be added later (as DHH was added alongside `kieran-rails-reviewer`). Decision needed before writing `kieran-go-reviewer.md`.

---

### Q2: Should `go-idioms` use a `references/` subdirectory, or be a single long SKILL.md?

`dhh-rails-style` uses 6 reference files. `dspy-ruby` is 700+ lines in a single SKILL.md. `andrew-kane-gem-writer` is ~180 lines with a small references/ dir.

For `go-idioms`, the proposed structure uses a `references/` dir with 4 files. This matches the `dhh-rails-style` model. The alternative is a single long SKILL.md (dspy-ruby model). The router-based `<intake>` / `<routing>` pattern used in `dhh-rails-style` requires the references subdirectory.

**Recommendation:** Use `references/` subdirectory for `go-idioms` (the broadest skill). Use single-file SKILL.md for `go-modules` and `go-testing` (narrower scope). Decision needed before writing go-idioms.

---

### Q3: Should `go-testing` reference `go-idioms` or be fully standalone?

Skills are consumed by agents and as direct skill invocations. If `go-testing` links to `go-idioms/references/patterns.md`, the skill becomes dependent on another skill's internal structure — a fragile coupling. Better to keep skills standalone with their own examples.

**Recommendation:** Skills should be standalone. Cross-references by name ("see go-idioms for package naming") are acceptable in prose but no hard file links across skill directories.

---

### Q4: Flat layout vs `go/` prefix for agent files?

The existing agent directories use categorical organization:
- `agents/review/kieran-rails-reviewer.md`
- `agents/review/kieran-python-reviewer.md`

A `go/` subdirectory is not used for any language. Options:
- **Option A:** Flat within category — `agents/review/kieran-go-reviewer.md` (matches existing pattern)
- **Option B:** Language subdirectory — `agents/review/go/kieran-go-reviewer.md` (would break convention, no precedent)

**Decision: Option A is the only correct approach** — follows the existing convention exactly. No subdirectory.

---

### Q5: Should `go-linter-advisor` extend `.golangci.yml` config, or only run existing config?

Two behaviors are possible:
1. Run whatever `.golangci.yml` exists (or defaults if absent) and report results
2. Also offer to create/upgrade `.golangci.yml` with an opinionated set of linters

Option 2 is more useful but also more destructive (modifying project config). The existing `lint` agent never modifies `.standardrb` config.

**Recommendation:** Default to run-only (Option 1). Agent can offer to generate a starter `.golangci.yml` only when no config file exists, and only after asking confirmation.

---

### Q6: Should the `setup` skill detection be for `go.mod` only, or also check for `*.go` files?

Projects without a `go.mod` (e.g., single-file scripts) still contain `.go` files but are not module-based projects. Most real projects have `go.mod`.

**Recommendation:** Detect on `go.mod` only. This is consistent with how Rails detection uses `config/routes.rb` (a distinctive file, not just "any .rb file").

---

### Q7: Minimum Go version to target in skill content?

Go has been stable for years but modules weren't the default until Go 1.16, generics arrived in 1.18, and `slog` arrived in 1.21.

**Recommendation:** Target Go 1.21+. Call out 1.18+ for fuzz testing in `go-testing`, 1.21+ for `slog` in `go-idioms`. Document the version assumption in a frontmatter comment in each skill.

---

### Q8: Should `go-concurrency` skill include `errgroup` and `semaphore` from `golang.org/x/sync`?

`golang.org/x/sync` is not in the standard library but is maintained by the Go team and nearly universally used. Including it in the skill content (vs. just stdlib sync) adds real value without adding a controversial third-party dependency.

**Recommendation:** Include `errgroup` and `semaphore` from `x/sync` in `go-concurrency`. Note that they require `go get golang.org/x/sync`.
