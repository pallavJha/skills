---
name: code-review
description: Review code changes for bugs, security issues, performance problems, and style. Use when the user asks for a code review, PR review, or wants feedback on their changes.
argument-hint: [file-or-directory]
allowed-tools: Read, Grep, Glob, Bash(git diff *), Bash(git log *), Bash(git show *)
---

Review the code specified by $ARGUMENTS. If no arguments are provided, review the current staged and unstaged changes (`git diff HEAD`).

## Review process

1. **Gather context**: Read the changed files and understand what the code does. Look at surrounding code for context. If reviewing a git diff, use `git diff --staged` as you're only supposed to review the staged chagnes. Mention that you'll only review the staged chagnes on start so that dev can stage the changes. 

2. **Check project rules**: If a `DEVELOPMENT.md` file exists in the project root, read it and verify that the code conforms to all rules, conventions, and guidelines defined there. Flag any violations as findings.

3. **Check language-specific rules**: Based on the language of the changed files, read the corresponding rules file from `${CLAUDE_SKILL_DIR}/` and verify the code conforms to all rules defined there. Flag any violations as findings.
   - Go: `${CLAUDE_SKILL_DIR}/go-rules.md`
   - JavaScript / TypeScript / Svelte: `${CLAUDE_SKILL_DIR}/js-rules.md`

4. **Analyze changes** across these dimensions, ordered by priority:

### Correctness
- Check boundary conditions in loops: `<` vs `<=`, zero-length inputs, single-element slices, empty maps. Off-by-one errors hide here.
- Check boolean logic under negation — `&&` vs `||` flips are easy to miss when a condition is inverted.
- Flag any dereference of a pointer, optional, or nullable value without a prior nil/undefined check.
- Flag type assertions without the `ok` check (Go) or unchecked casts (TypeScript). These panic or produce undefined behavior silently.
- Flag functions that return both a value and an error where the caller only checks one.
- Check that function arguments are passed in the correct order, especially when multiple arguments share the same type (e.g., `copy(dst, src)` vs `copy(src, dst)`).
- Flag return values that are ignored or misinterpreted — e.g., treating a count as a boolean, or ignoring a second return value.
- Flag callers that assume ordering, uniqueness, or idempotency that the callee doesn't guarantee.

### Security
- Flag any user-controlled input passed into SQL queries, shell commands, HTML templates, or file paths without sanitization. This includes indirect paths — e.g., a URL parameter that flows through two functions before hitting a query.
- Flag hardcoded secrets, API keys, tokens, or credentials. Check for them in config files, environment variable defaults, and test fixtures.
- Flag sensitive data (passwords, PII, tokens) that appears in log statements, error messages, or HTTP responses.
- Flag missing authentication or authorization checks on endpoints or functions that modify state or return sensitive data.
- Flag uses of `unsafe`, `eval`, `exec`, `dangerouslySetInnerHTML`, raw SQL string interpolation, or deserialization of untrusted input.
- Flag file operations that accept user-controlled paths without normalizing or restricting to an expected directory (path traversal).
- Flag overly permissive CORS, cookie, or header configurations.

### Bugs & reliability
- Flag errors that are silently discarded — `_ = fn()`, empty `catch {}`, or error returns that are never checked by the caller.
- Flag bare error returns (`return err`) that lose context. Errors should be wrapped with what the caller was trying to do.
- Flag resource acquisitions (file handles, DB connections, HTTP bodies, locks) without a corresponding close/release, ideally via `defer` or `finally`.
- Flag `defer` or `finally` blocks that silently swallow close/cleanup errors.
- Flag panics, `os.Exit`, or `process.exit` in library code or request handlers — these kill the process instead of propagating the error.
- Flag retry logic without backoff, or retry logic that retries non-idempotent operations.
- Flag timeout-sensitive operations (network calls, DB queries) that don't use a context or deadline.

### Performance
- Flag allocations inside tight loops — object creation, slice appends without pre-allocation, string concatenation in a loop, or repeated map lookups that could be hoisted.
- Flag N+1 query patterns: a query inside a loop where a single batch query or join would work.
- Flag blocking/synchronous calls (file I/O, network, sleep) inside async contexts or on the main thread.
- Flag algorithms with O(n²) or worse complexity when the input size is unbounded or large. Look for nested loops over the same collection, repeated linear searches, or sorts inside loops.
- Flag unnecessary copies of large structs or buffers — passing by pointer or using slices/views is often sufficient.
- Flag missing indexes on columns used in `WHERE`, `JOIN`, or `ORDER BY` clauses in new or modified queries.

### Comments & readability
- Flag verbose, redundant, or boilerplate comments. Comments are a maintenance burden — every comment is a liability that can drift from the code it describes.
- A comment is only justified when the code *cannot* be made self-explanatory through better naming, simpler structure, or smaller functions. If the fix is to rewrite the code, suggest that instead of keeping the comment.
- Flag comments that restate what the code does (e.g., `// increment counter` above `counter++`). The code already says that.
- Flag doc comments that are longer than necessary. One line is the default. Multi-line only when there's a genuine non-obvious constraint or gotcha.
- Never suggest adding comments. If code is unclear, suggest making the code clearer.

### Design & maintainability
- Flag functions longer than ~50 lines or with more than 3 levels of nesting. These are candidates for extraction — not as a style preference, but because deep nesting hides bugs.
- Flag functions that take more than 4–5 parameters. This usually means a struct/options object is needed, or the function is doing too much.
- Flag boolean parameters that change function behavior — these produce code that's unreadable at the call site (e.g., `process(true, false)`). Suggest separate functions or an options struct.
- Flag violations of existing project conventions visible in surrounding code — naming patterns, error handling style, module structure. New code should look like it belongs.
- Flag code that mixes abstraction levels in the same function — e.g., HTTP header parsing next to business logic next to database calls.
- Flag dead code, unreachable branches, or conditions that are always true/false.

## Output format

Structure your review as follows:

**Summary**: One sentence describing what the changes do.

**Findings**: List issues grouped by severity:

- **Critical**: Must fix before merging (bugs, security, data loss risks)
- **Warning**: Should fix, likely to cause problems (performance, reliability)
- **Suggestion**: Optional improvements (readability, style, minor simplifications)

For each finding, include:
- The file and line number
- What the issue is
- Why it matters
- A concrete fix (code snippet when helpful)

**Verdict**: One of:
- **Approve** - No critical or warning issues found
- **Request changes** - Critical or warning issues that should be addressed

If the code looks good and you find no issues, say so clearly rather than inventing nitpicks.
