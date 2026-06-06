### Don't write complex one-line conditionals

Go doesn't have ternaries, but the same readability problem shows up when conditions get crammed into a single expression or a dense one-liner. Prefer plain `if`/`else`.

**Bad:**
```go
const maxUTCOffsetMinutes = 14 * 60

utcOffset := 0
if v, ok := req.Body["utcOffset"].(int); ok && v >= -maxUTCOffsetMinutes && v <= maxUTCOffsetMinutes { utcOffset = v }
```

**Good:**
```go
const maxUTCOffsetMinutes = 14 * 60

utcOffset := 0
v, ok := req.Body["utcOffset"].(int)
if ok && v >= -maxUTCOffsetMinutes && v <= maxUTCOffsetMinutes {
    utcOffset = v
}
```

### Don't decide by yourself

- You can just make a decision on your own. For example, if there's a Go module in a monorepo which is not part of the root `go.work` file and
  you've been tasked with wiring it up, don't assume you should add it to the `use (...)` block in `go.work`. Ask with nice options.

### Use proper types

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

### Handle errors explicitly

Never discard an error with `_` unless you have a concrete reason, and never let an error fall through silently. Check it where it happens and either handle it or wrap it with context using `fmt.Errorf("...: %w", err)`.

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

### Compare errors with `errors.Is` / `errors.As`

Never use `==` to compare errors or type-assert directly — wrapped errors (`%w`) won't match. Use `errors.Is` for sentinel values and `errors.As` for typed errors.

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

### `context.Context` is the first argument

Always pass `ctx context.Context` as the first parameter of any function that does I/O, blocks, or calls other context-aware code. Never store a `Context` in a struct field — pass it through the call chain.

```go
// Bad
func (s *Server) Fetch(id string, ctx context.Context) (*User, error)
type Worker struct { ctx context.Context } // don't do this

// Good
func (s *Server) Fetch(ctx context.Context, id string) (*User, error)
```

### Every goroutine needs a known exit

Don't `go f()` without knowing how it stops. A goroutine that blocks forever on a channel or a network call is a silent leak — no panic, no log. Tie its lifetime to a `context.Context`, a `done` channel, or a `sync.WaitGroup` the caller controls.

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

### Don't put non-trivial work in `init()`

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

### Do not become the Co Committer
You will never be responsible for the changes you make, so never become a Co Committer.

### Don't use `!= ""` check for determining if the string is empty
```go
// Bad
if flags.LogLevel == "" {

}

// Good
if len(flags.LogLevel) == 0 {

}
```

### Error message should contain the intent for which the failing function was called
```go
// Bad
defer func() {
  if err := rdb.Close(); err != nil {
    ctx.L().Warn().Err(err).Msg("redis close")
  }
}()

// Good
defer func() {
  if err := rdb.Close(); err != nil {
    ctx.L().Warn().Err(err).Msg("error while closing the redis client")
  }
}()
```

### Use `fmt.Sprintf` instead of string concatenation
```go
// Bad
return l.allow(ctx, "rate:ip:"+ip, l.cfg.PerIPRateLimit)

// Good
return l.allow(ctx, fmt.Sprintf("rate:ip:%s", ip), l.cfg.PerIPRateLimit)
```

### Do not write oneliner return statement
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

### Keep doc comments short

Doc comments on exported identifiers default to one line. Multi-line only when the WHY is non-obvious (hidden constraint, workaround, surprising behavior). Never describe what the code does — the code already does that.

```go
// Bad — reads like a magazine article.
// SMTPCodeMailboxUnavailable is the permanent "requested action not taken:
// mailbox unavailable" reply. Returned when the recipient domain is not in
// the configured allowlist. Defined as a named constant for visibility at
// the call site.
const SMTPCodeMailboxUnavailable = 550

// Good — one line, references the spec.
// SMTPCodeMailboxUnavailable: RFC 5321 §4.2.3 "mailbox unavailable".
const SMTPCodeMailboxUnavailable = 550
```

### Handle Close errors in tests

A deferred Close in a test must report the error via t.Errorf. Silent drops hide leaks and flaky teardown.

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

### Constant names start with a capital letter

Constants — `const` declarations and `var` declarations that hold values intended to be immutable for the program's lifetime (sentinel errors, precompiled scripts, lookup tables, named codes) — start with a capital letter even when only used within the same package. This keeps named-constant declarations visually distinct from runtime variables and makes intent obvious to readers.

```go
// Bad
const smtpCodeMailboxUnavailable = 550
var errRecipientDomainNotAccepted = &gosmtp.SMTPError{...}

// Good
const SMTPCodeMailboxUnavailable = 550
var ErrRecipientDomainNotAccepted = &gosmtp.SMTPError{...}
```

### Don't create variables with poor names

A variable's name is its only documentation at the call site. A reader scanning the function should know what each name refers to without scrolling back. Three failure modes to avoid:

1. **Truncated-word abbreviations** (`rl`, `st`, `cErr`, `sCancel`, `gen`, `rs`). Save no real space, lose all meaning. Use the word.
2. **Single-letter locals outside tight idioms.** `i` in a loop, `g`/`s` as a receiver, `w`/`r` in an `http.HandlerFunc` are fine because the idiom carries the meaning. `t` for a `time.Time` or `c` for a "client" is not.
3. **Generic placeholders** (`data`, `result`, `value`, `item`). Pick a name that says *what* the thing is — `payload`, `allocated`, `traceID`, `candidate`.

```go
// Bad
rl := ratelimit.New(rdb, cfg.Limits)
st := store.New(rdb, cfg)
rs := redsync.New(goredislib.NewPool(rdb))
gen := NewEmailGenerator(rdb, domains)

defer func() {
    if cErr := rdb.Close(); cErr != nil {
        ctx.L().Warn().Err(cErr).Msg("error while closing the redis client")
    }
}()
shutdownCtx, sCancel := context.WithTimeout(ctx, ShutdownTimeout)
defer sCancel()

// Good
rateLimiter := ratelimit.New(rdb, cfg.Limits)
store := store.New(rdb, cfg)
locker := redsync.New(goredislib.NewPool(rdb))
emailGen := NewEmailGenerator(rdb, domains)

defer func() {
    if closeErr := rdb.Close(); closeErr != nil {
        ctx.L().Warn().Err(closeErr).Msg("error while closing the redis client")
    }
}()
shutdownCtx, shutdownCancel := context.WithTimeout(ctx, ShutdownTimeout)
defer shutdownCancel()
```

Exceptions stay narrow: `ctx`, `cfg`, `rdb`, `err`, `mu`, `wg`, `ch`, struct receivers — these are universal enough that readers parse them as the full word automatically. Everything else gets spelled out.
