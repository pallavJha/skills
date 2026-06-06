# code-review

Claude Code skill that reviews staged changes. Checks correctness, security, bugs, performance, comment bloat, and design. Reads your project's `DEVELOPMENT.md` and language-specific rule files so it reviews against your conventions.

## Setup

### Claude Code

```bash
# all projects
ln -s /path/to/skills/code-review ~/.claude/skills/code-review

# single project
cp -r code-review your-project/.claude/skills/code-review
```

### Codex

```bash
# all projects
ln -s /path/to/skills/code-review ~/.agents/skills/code-review

# single repo
ln -s /path/to/skills/code-review your-repo/.agents/skills/code-review
```

Both read the same `SKILL.md` format.

## Usage

```bash
/code-review                # staged changes
/code-review src/auth.ts    # specific file
```

Also triggers automatically when you ask Claude to review code.

## What it checks

- **Correctness** — nil dereference without check, off-by-one, swapped args, ignored return values
- **Security** — unsanitized input in SQL/shell/HTML, hardcoded secrets, PII in logs, missing auth, path traversal
- **Bugs** — swallowed errors, bare `return err`, resource leaks, missing timeouts
- **Performance** — allocations in loops, N+1 queries, O(n²) on unbounded input, blocking in async
- **Comments** — flags bloat, never suggests adding comments
- **Design** — functions >50 lines, >3 nesting levels, boolean flag params, mixed abstraction levels, dead code

Output: file, line, issue, why, fix. Verdict is **Approve** or **Request changes**.

## Language rules

```
code-review/
├── SKILL.md
├── go-rules.md
├── js-rules.md
└── README.md
```

Skill reads the matching rule file based on language. Add more and update step 3 in `SKILL.md`.

