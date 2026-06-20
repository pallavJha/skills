---
name: code-review
description: Review code changes for bugs, security issues, performance problems, and style. Use when the user asks for a code review, PR review, or wants feedback on their changes.
argument-hint: [file-or-directory]
allowed-tools: Read, Grep, Glob, Agent, Bash(git diff *), Bash(git log *), Bash(git show *)
---

Review the code specified by $ARGUMENTS. If no arguments are provided, review the current staged and unstaged changes (`git diff HEAD`).

## Review process

Do NOT review by reading the diff once and writing down what stands out — that misses rules. Follow the steps below in order.

1. **Gather context**: Read the changed files and understand what the code does. Look at surrounding code for context. If reviewing a git diff, use `git diff --staged` as you're only supposed to review the staged changes. Mention that you'll only review the staged changes on start so that dev can stage the changes.

2. **Build the rule inventory**: Collect every rule that applies to this review. Rule IDs are written in the source files — never invent or renumber them:
   - `D1..Dn`: rules from the project root's `DEVELOPMENT.md`, if it exists. This file has no embedded IDs, so number its rules at review time in the order they appear.
   - `JS#` / `GO#` / `HTML#`: rules from the language rules files in `${CLAUDE_SKILL_DIR}/` matching the changed files' languages. Each rule's ID is in its heading (e.g. `### JS10 — No complex ternaries`).
     - Go: `${CLAUDE_SKILL_DIR}/go-rules.md`
     - JavaScript / TypeScript / Svelte: `${CLAUDE_SKILL_DIR}/js-rules.md`
     - HTML / Svelte / JSX markup: `${CLAUDE_SKILL_DIR}/html-rules.md`
   - `C#`/`S#`/`B#`/`P#`/`R#`/`M#`: the dimension checks below; each bullet carries its ID.

   The inventory is the contract for the review — every ID must receive a verdict before the review can end.

3. **Grep pre-pass**: Mechanically detectable rules carry a `Grep:` line in the rules files, listing one or more regex patterns in backticks. Run every pattern against the changed files only (Grep tool or `grep -nE`). Each hit is a candidate violation: during the sweep it must be either confirmed as a finding or dismissed with a one-line reason (e.g. "JS10 hit at api.ts:42 — `?.` optional chaining, not a ternary"). Never silently drop a hit.

   The `Grep:` patterns are the ONLY things you search for. The code examples inside rules exist to explain the rule — never grep for identifiers, strings, or values taken from an example (e.g. do not search for `HttpHeaders`, `x-some-header`, or `consumeDirty`). Finding zero example keywords proves nothing about the rule.

4. **Parallel sweep**: Split the inventory into four slices and review them concurrently — spawn all four agents in a single message so they actually run in parallel:
   - Agent 1: `D#` + the language rules (`JS#` / `GO#` / `HTML#`)
   - Agent 2: `C#` + `B#` (correctness and reliability — the deepest read)
   - Agent 3: `S#` + `P#` (security and performance)
   - Agent 4: `R#` + `M#` (readability and maintainability)

   Each agent's prompt must contain:
   - The changed file paths and the exact `git diff` command for the scope under review.
   - Which rules file(s) to read and the exact rule IDs it owns.
   - The grep candidates belonging to its rules (from step 3) — agents don't re-grep.
   - The sweep rules verbatim: check every owned rule against every changed file by reading the changed code; rule examples are illustrative, never searched for; zero grep hits never makes a rule clean.
   - The required return format — raw data, not prose: per rule ID a verdict (`clean`, `n/a` + reason, or violations as `file:line`), findings at ≤3 sentences each, and one-line candidate dismissals.

   Exception: for small diffs (under ~50 changed lines), skip the agents and sweep the whole inventory inline — spawn overhead would dominate the savings.

5. **Merge & verify** (in the main conversation, after all agents return):
   - Merge the agent results. Dedupe findings caught by two slices — keep the better-argued one and list both rule IDs on it.
   - Audit coverage: every inventory ID must have a verdict from its agent; any missing ID gets checked now, inline.
   - Re-check the grep candidates: each one is either a finding or has a dismissal reason.
   - Adversarial re-read: read the full diff once more asking only "which rule violations did every agent miss?"

   This step may add findings and fill gaps. It must never remove an agent's finding — downgrading severity with a stated reason is allowed.

## Dimension checks

### Correctness
- **C1** Check boundary conditions in loops: `<` vs `<=`, zero-length inputs, single-element slices, empty maps. Off-by-one errors hide here.
- **C2** Check boolean logic under negation — `&&` vs `||` flips are easy to miss when a condition is inverted.
- **C3** Flag any dereference of a pointer, optional, or nullable value without a prior nil/undefined check.
- **C4** Flag type assertions without the `ok` check (Go) or unchecked casts (TypeScript). These panic or produce undefined behavior silently.
- **C5** Flag functions that return both a value and an error where the caller only checks one.
- **C6** Check that function arguments are passed in the correct order, especially when multiple arguments share the same type (e.g., `copy(dst, src)` vs `copy(src, dst)`).
- **C7** Flag return values that are ignored or misinterpreted — e.g., treating a count as a boolean, or ignoring a second return value.
- **C8** Flag callers that assume ordering, uniqueness, or idempotency that the callee doesn't guarantee.

### Security
- **S1** Flag any user-controlled input passed into SQL queries, shell commands, HTML templates, or file paths without sanitization. This includes indirect paths — e.g., a URL parameter that flows through two functions before hitting a query.
- **S2** Flag hardcoded secrets, API keys, tokens, or credentials. Check for them in config files, environment variable defaults, and test fixtures.
- **S3** Flag sensitive data (passwords, PII, tokens) that appears in log statements, error messages, or HTTP responses.
- **S4** Flag missing authentication or authorization checks on endpoints or functions that modify state or return sensitive data.
- **S5** Flag uses of `unsafe`, `eval`, `exec`, `dangerouslySetInnerHTML`, raw SQL string interpolation, or deserialization of untrusted input.
- **S6** Flag file operations that accept user-controlled paths without normalizing or restricting to an expected directory (path traversal).
- **S7** Flag overly permissive CORS, cookie, or header configurations.

### Bugs & reliability
- **B1** Flag errors that are silently discarded — `_ = fn()`, empty `catch {}`, or error returns that are never checked by the caller.
- **B2** Flag bare error returns (`return err`) that lose context. Errors should be wrapped with what the caller was trying to do.
- **B3** Flag resource acquisitions (file handles, DB connections, HTTP bodies, locks) without a corresponding close/release, ideally via `defer` or `finally`.
- **B4** Flag `defer` or `finally` blocks that silently swallow close/cleanup errors.
- **B5** Flag panics, `os.Exit`, or `process.exit` in library code or request handlers — these kill the process instead of propagating the error.
- **B6** Flag retry logic without backoff, or retry logic that retries non-idempotent operations.
- **B7** Flag timeout-sensitive operations (network calls, DB queries) that don't use a context or deadline.

### Performance
- **P1** Flag allocations inside tight loops — object creation, slice appends without pre-allocation, string concatenation in a loop, or repeated map lookups that could be hoisted.
- **P2** Flag N+1 query patterns: a query inside a loop where a single batch query or join would work.
- **P3** Flag blocking/synchronous calls (file I/O, network, sleep) inside async contexts or on the main thread.
- **P4** Flag algorithms with O(n²) or worse complexity when the input size is unbounded or large. Look for nested loops over the same collection, repeated linear searches, or sorts inside loops.
- **P5** Flag unnecessary copies of large structs or buffers — passing by pointer or using slices/views is often sufficient.
- **P6** Flag missing indexes on columns used in `WHERE`, `JOIN`, or `ORDER BY` clauses in new or modified queries.

### Comments & readability
- **R1** Flag verbose, redundant, or boilerplate comments. Comments are a maintenance burden — every comment is a liability that can drift from the code it describes.
- **R2** A comment is only justified when the code *cannot* be made self-explanatory through better naming, simpler structure, or smaller functions. If the fix is to rewrite the code, suggest that instead of keeping the comment.
- **R3** Flag comments that restate what the code does (e.g., `// increment counter` above `counter++`). The code already says that.
- **R4** Flag doc comments that are longer than necessary. One line is the default. Multi-line only when there's a genuine non-obvious constraint or gotcha.
- **R5** Never suggest adding comments. If code is unclear, suggest making the code clearer.

### Design & maintainability
- **M1** Flag functions longer than ~50 lines or with more than 3 levels of nesting. These are candidates for extraction — not as a style preference, but because deep nesting hides bugs.
- **M2** Flag functions that take more than 4–5 parameters. This usually means a struct/options object is needed, or the function is doing too much.
- **M3** Flag boolean parameters that change function behavior — these produce code that's unreadable at the call site (e.g., `process(true, false)`). Suggest separate functions or an options struct.
- **M4** Flag violations of existing project conventions visible in surrounding code — naming patterns, error handling style, module structure. New code should look like it belongs.
- **M5** Flag code that mixes abstraction levels in the same function — e.g., HTTP header parsing next to business logic next to database calls.
- **M6** Flag dead code, unreachable branches, or conditions that are always true/false.

## Output format

The report contains findings, coverage, and a verdict — nothing else. No narration: don't describe your process, don't praise the code, don't list the invariants you verified. The sweep and verify passes are work you do, not content you report.

Structure your review as follows:

**Summary**: One sentence describing what the changes do.

**Findings**: List issues grouped by severity:

- **Critical**: Must fix before merging (bugs, security, data loss risks)
- **Warning**: Should fix, likely to cause problems (performance, reliability)
- **Suggestion**: Optional improvements (readability, style, minor simplifications)

For each finding, include the rule ID it violates (or `—` for issues outside the inventory), the file and line number, and at most three sentences: the issue, why it matters, the fix. Add a code snippet only when the fix isn't clear from one line. For a style rule violated at many sites (e.g. braces), one finding with the line list — not one per site.

**Grep dismissals**: One line per dismissed candidate. Skip this section when there are none.

**Coverage**: Every inventory ID appears exactly once. Only rules with findings get their own row; collapse everything else into two list lines without per-rule justifications:

| Rule | Result |
|------|--------|
| JS9 braces | 17 findings (#4) |
| JS10 no complex ternaries | 2 findings (#5) |

Clean: JS1 JS2 JS8 JS13 C1–C8 S1–S7 B1 P1 P3 P4 R2 R4 M2–M5
n/a: D (no DEVELOPMENT.md), JS4–JS7 JS17 JS19–JS21 C4 C5 B2–B7 P2 P5 P6

A missing ID means the review is incomplete — go back to the verify pass.

**Verdict**: One of:
- **Approve** - No critical or warning issues found
- **Request changes** - Critical or warning issues that should be addressed

If the code looks good and you find no issues, say so clearly rather than inventing nitpicks.
