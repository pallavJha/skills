### GO1 — Don't write complex one-line conditionals

Go doesn't have ternaries, but the same readability problem shows up when conditions get crammed into a single expression or a dense one-liner. Prefer plain `if`/`else`.

Grep: `^\s*if .*\{.*\}\s*$`

**Bad:**
```go
const maxPageSize = 100

pageSize := defaultPageSize
if v, ok := req.Body["pageSize"].(int); ok && v > 0 && v <= maxPageSize { pageSize = v }
```

**Good:**
```go
const maxPageSize = 100

pageSize := defaultPageSize
v, ok := req.Body["pageSize"].(int)
if ok && v > 0 && v <= maxPageSize {
    pageSize = v
}
```

### GO2 — Don't decide by yourself

- You can just make a decision on your own. For example, if there's a Go module in a monorepo which is not part of the root `go.work` file and
  you've been tasked with wiring it up, don't assume you should add it to the `use (...)` block in `go.work`. Ask with nice options.

### GO3 — Use proper types

Always use explicit, concrete types. Avoid `interface{}` / `any` unless genuinely required. If `any` is needed, narrow it with a type assertion or type switch before use.

```go
// Bad
func process(data any) string {
    return data.(map[string]any)["name"].(string)
}

// Good
type User struct {
    Name string
    Age  int
}

func process(u User) string {
    return u.Name
}
```

### GO4 — Handle errors explicitly

Never discard an error with `_` unless you have a concrete reason, and never let an error fall through silently. Check it where it happens and either handle it or wrap it with context using `fmt.Errorf("...: %w", err)`.

Grep: `(^|, |\s)_ :?=`

```go
// Bad
data, _ := os.ReadFile(path)
process(data)

// Bad
data, err := os.ReadFile(path)
if err != nil {
    return err
}

// Good
data, err := os.ReadFile(path)
if err != nil {
    return fmt.Errorf("cannot read config %s: %w", path, err)
}
```

### GO5 — Compare errors with `errors.Is` / `errors.As`

Never use `==` to compare errors or type-assert directly — wrapped errors (`%w`) won't match. Use `errors.Is` for sentinel values and `errors.As` for typed errors.

Grep: `err (==|!=) [^n]` , `err\.\(`

```go
// Bad
if err == os.ErrNotExist { ... }
if pathErr, ok := err.(*os.PathError); ok { ... }

// Good
if errors.Is(err, os.ErrNotExist) {
    ...
}
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    ...
}
```

### GO6 — `context.Context` is the first argument

Always pass `ctx context.Context` as the first parameter of any function that does I/O, blocks, or calls other context-aware code. Never store a `Context` in a struct field — pass it through the call chain.

Grep: `, ctx context\.Context`

```go
// Bad
func (s *Server) Fetch(id string, ctx context.Context) (*User, error)
type Worker struct { ctx context.Context } // don't do this

// Good
func (s *Server) Fetch(ctx context.Context, id string) (*User, error)
```

### GO7 — Every goroutine needs a known exit

Don't `go f()` without knowing how it stops. A goroutine that blocks forever on a channel or a network call leaks silently, with no panic and no log to point at it. Tie its lifetime to a `context.Context`, a `done` channel, or a `sync.WaitGroup` the caller controls.

```go
// Bad — leaks if the receiver never reads
go func() {
    results <- doWork()
}()

// Good — exits when ctx is cancelled
go func() {
    select {
    case results <- doWork():
    case <-ctx.Done():
    }
}()
```

### GO8 — Don't put non-trivial work in `init()`

`init()` runs implicitly at import time, in an order you don't fully control, and can't be disabled or mocked in tests. Keep it to trivial registration (e.g. `sql.Register`). Anything that can fail, do I/O, or read config belongs in an explicit `New...()` constructor the caller invokes.

```go
// Bad
func init() {
    db = mustConnect(os.Getenv("DATABASE_URL"))
}

// Good
func NewStore(ctx context.Context, dsn string) (*Store, error) {
    db, err := connect(ctx, dsn)
    if err != nil {
        return nil, fmt.Errorf("connect %s: %w", dsn, err)
    }
    return &Store{db: db}, nil
}
```

### GO9 — Do not become the Co Committer
You will never be responsible for the changes you make, so never become a Co Committer.

### GO10 — Don't use `!= ""` check for determining if the string is empty

Grep: `(==|!=) ""`

```go
// Bad
if flags.LogLevel == "" {

}

// Good
if len(flags.LogLevel) == 0 {

}
```

### GO11 — Error message should contain the intent for which the failing function was called
```go
// Bad
defer func() {
  if err := f.Close(); err != nil {
    ctx.L().Warn().Err(err).Msg("file close")
  }
}()

// Good
defer func() {
  if err := f.Close(); err != nil {
    ctx.L().Warn().Err(err).Msg("error while closing the export file")
  }
}()
```

### GO12 — Use `fmt.Sprintf` instead of string concatenation

Grep: `" ?\+ ?[[:alnum:]_]` , `[[:alnum:]_)] ?\+ ?"`

```go
// Bad
return store.get(ctx, "user:"+userID, cfg.CacheTTL)

// Good
return store.get(ctx, fmt.Sprintf("user:%s", userID), cfg.CacheTTL)
```

### GO13 — Do not write oneliner return statement

Grep: `\{ return .* \}`

```go
// Bad
func (c *Context) Done() <-chan struct{} { return c.ctx.Done() }
func (c *Context) Err() error            { return c.ctx.Err() }
func (c *Context) Value(key any) any     { return c.ctx.Value(key) }

// Good
func (c *Context) Done() <-chan struct{} {
	return c.ctx.Done()
}
func (c *Context) Err() error {
	return c.ctx.Err()
}
func (c *Context) Value(key any) any {
	return c.ctx.Value(key)
}
```

### GO14 — Keep doc comments short

Doc comments on exported identifiers default to one line. Multi-line only when the WHY is non-obvious (hidden constraint, workaround, surprising behavior). Never describe what the code does — the code already does that.

```go
// Bad — reads like a magazine article.
// HTTPStatusTooManyRequests is the reply sent when the client has issued more
// requests than the configured window allows. Returned once the caller trips
// the rate limiter. Defined as a named constant for visibility at the call
// site.
const HTTPStatusTooManyRequests = 429

// Good — one line, references the spec.
// HTTPStatusTooManyRequests: RFC 6585 §4 "Too Many Requests".
const HTTPStatusTooManyRequests = 429
```

### GO15 — Handle Close errors in tests

A deferred Close in a test must report the error via t.Errorf. Silent drops hide leaks and flaky teardown.

Grep: `_ = .*\.Close\(\)`

```go
// Bad
defer func() { _ = client.Close() }()

// Good
defer func() {
	if err := client.Close(); err != nil {
		t.Errorf("client.Close: %v", err)
	}
}()
```

### GO16 — Constant names start with a capital letter

Constants — `const` declarations and `var` declarations that hold values intended to be immutable for the program's lifetime (sentinel errors, precompiled scripts, lookup tables, named codes) — start with a capital letter even when only used within the same package. This keeps named-constant declarations visually distinct from runtime variables and makes intent obvious to readers.

Grep: `const [a-z]`

```go
// Bad
const httpStatusTooManyRequests = 429
var errRequestQuotaExceeded = &apierr.APIError{...}

// Good
const HTTPStatusTooManyRequests = 429
var ErrRequestQuotaExceeded = &apierr.APIError{...}
```

### GO17 — Don't create variables with poor names

A variable's name is its only documentation at the call site. A reader scanning the function should know what each name refers to without scrolling back. Three failure modes to avoid:

1. **Truncated-word abbreviations** (`lim`, `st`, `cErr`, `sCancel`, `gen`, `mtx`). Save no real space, lose all meaning. Use the word.
2. **Single-letter locals outside tight idioms.** `i` in a loop, `g`/`s` as a receiver, `w`/`r` in an `http.HandlerFunc` are fine because the idiom carries the meaning. `t` for a `time.Time` or `c` for a "client" is not.
3. **Generic placeholders** (`data`, `result`, `value`, `item`). Pick a name that says *what* the thing is — `payload`, `normalized`, `traceID`, `candidate`.

```go
// Bad
lim := ratelimit.New(client, cfg.Limits)
st := store.New(client, cfg)
mtx := mutex.New(client)
gen := NewIDGenerator(client, cfg.IDs)

defer func() {
    if cErr := client.Close(); cErr != nil {
        ctx.L().Warn().Err(cErr).Msg("error while closing the cache client")
    }
}()
shutdownCtx, sCancel := context.WithTimeout(ctx, ShutdownTimeout)
defer sCancel()

// Good
rateLimiter := ratelimit.New(client, cfg.Limits)
store := store.New(client, cfg)
locker := mutex.New(client)
idGenerator := NewIDGenerator(client, cfg.IDs)

defer func() {
    if closeErr := client.Close(); closeErr != nil {
        ctx.L().Warn().Err(closeErr).Msg("error while closing the cache client")
    }
}()
shutdownCtx, shutdownCancel := context.WithTimeout(ctx, ShutdownTimeout)
defer shutdownCancel()
```

Exceptions stay narrow: `ctx`, `cfg`, `db`, `err`, `mu`, `wg`, `ch`, struct receivers — these are universal enough that readers parse them as the full word automatically. Everything else gets spelled out.

### GO18 — Package names are the natural short word
Don't suffix package names to avoid collisions with the standard library; alias at the import site instead.

```go
// Bad
package httpsrv

// Good
package http
```

### GO19 — Group constants per type with a type prefix
When several domains share a result/status concept, give each its own prefixed constants instead of one shared generic set.

```go
// Bad
const ResultPass = "pass" // shared by lint, build, test

// Good
const LintResultPass = "pass"
const BuildResultPass = "pass"
```

### GO20 — Functions receive the config object, not the config path
Passing a path forces every test through the same file. Passing the object lets test setup read the config once and mutate it per testcase (e.g. toggling TLS).

```go
// Bad
func Run(configPath string) error

// Good
func Run(ctx context.Context, cfg *Config) error
```

### GO21 — `t.Helper()` belongs only in true helpers
A testcase function calling `t.Helper()` is wrong — it would hide the testcase itself from failure traces. Only shared assertion/setup helpers call it.

Grep: `t\.Helper\(\)`

### GO22 — Hide test-only constructs behind build tags
Constructors, resolver overrides, and hooks that exist only for tests must be unreachable from production code — guard them with `//go:build test` (or equivalent) rather than relying on convention.

### GO23 — Prefer templates over near-duplicate string builders
Two functions that differ only in content type (or one format detail) are one function with a parameter — or a `text/template`. Let the specific one delegate to the general one.
