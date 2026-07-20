# ai-codereviewer-skill

Slash commands for running parallel, severity-ranked code reviews and turning the results into fixed, PR'd code.

- **Claude Code users:** use `.claude/commands/` — `/cr-run`, `/cr-sec`, `/cr-recheck`, and `/cr-fix` work out of the box with auto-discovery.
- **Any other tool** (Cursor, Aider, Codex, ChatGPT, etc.): use the plain-text prompts in [`prompts/`](prompts/) — copy-paste `prompts/cr-run.md`, `prompts/cr-sec.md`, `prompts/cr-recheck.md`, or `prompts/cr-fix.md`, filling in the `{{LEVEL}}` / `{{ISSUES}}` placeholder, into your tool of choice. Same logic, no Claude Code-specific syntax.

## Commands

### `/cr-run [low|medium|high]`

Runs a read-only code review using parallel subagents. Never edits files.

- `low` — 1 subagent, quick pass for obvious bugs.
- `medium` (default) — 3 subagents, split by lens: correctness, performance/dead code, security/error-handling.
- `high` — 5+ subagents, split by lens and by codebase area for deep coverage.

Findings are ranked and tagged with a severity scale:

| Emoji | Level | Meaning |
|---|---|---|
| 🔴 | Critical | crashes, data loss, security vuln, broken core functionality |
| 🟠 | High | clear user-facing bug |
| 🟡 | Medium | logic error / meaningful perf issue / maintainability hazard |
| 🟢 | Low | minor inefficiency, dead code, unclear error handling |
| ⚪ | Trivial | style/naming, no functional impact |

After reporting findings, Claude asks whether to file them as GitHub issues via `gh issue create` — it will not file issues without confirmation.

**Usage:**
```
/cr-run
/cr-run high
```

### `/cr-sec [low|medium|high]`

Runs a read-only **security** review using parallel subagents — the security-only counterpart to `/cr-run`. Never edits files.

- `low` — 1 subagent, quick pass for high-confidence, high-impact vulnerabilities.
- `medium` (default) — 3 subagents, split by category group: input validation & injection, auth/crypto/secrets, code execution & data exposure.
- `high` — 5+ subagents, split by category and by codebase area.

Covers SQL/command/template/NoSQL injection, path traversal, auth bypass, privilege escalation, JWT/session flaws, hardcoded secrets, weak crypto, insecure deserialization, RCE, XSS, and sensitive data exposure — with a curated exclusion list (DoS, outdated deps, theoretical race conditions, etc.) and an 8/10+ confidence bar to keep noise down. Same GitHub-issue-filing flow as `/cr-run`, tagged `[security]`.

The vulnerability taxonomy, severity/confidence scoring, and exclusion list are adapted from Anthropic's open-source [`claude-code-security-review`](https://github.com/anthropics/claude-code-security-review) GitHub Action (MIT licensed) — full credit to that project for the underlying security review methodology.

**Usage:**
```
/cr-sec
/cr-sec high
```

### `/cr-recheck [issue-number(s)|all]`

Re-verifies open issues against the *current* code before you spend fix effort on them. If you file an issue, keep coding, and never get around to fixing it, the surrounding code may have already changed — the bug might be gone, moved, or refactored away. Read-only with respect to code; only reads source and updates/closes issues.

- Re-reads each issue's `file:line` and surrounding context fresh — doesn't trust the recorded line number.
- **confirmed** — still valid, left open (with a corrected line number comment if it moved).
- **stale** — no longer applies (already fixed/refactored/deleted); closed with a comment explaining why, so it's not re-investigated later.
- **needs-human** — ambiguous from reading code alone; left open and flagged.

Run this before `/cr-fix` on any batch of older issues to avoid wasting tokens re-diagnosing things that already resolved themselves.

**Usage:**
```
/cr-recheck
/cr-recheck 42 43
```

### `/cr-fix <issue-number(s)|all>`

Fixes GitHub issue(s), typically ones filed by `/cr-run`, and opens a pull request. This command does edit code.

- Re-verifies each issue is still valid before touching anything (skips stale/already-fixed ones) — for a larger batch of older issues, run `/cr-recheck` first instead of relying on this per-issue check.
- Fixes all targeted issues on a single branch (no per-issue or per-subagent branches).
- Runs verification (tests / exercising the code path) before committing.
- Opens one PR with a title of your choosing and a description listing every issue resolved (`Fixes #<n>` per issue) so GitHub auto-links/closes them.

**Usage:**
```
/cr-fix 42
/cr-fix 42 43
/cr-fix all
```

## Requirements

- [GitHub CLI](https://cli.github.com/) (`gh`) authenticated for the target repo, for issue/PR creation.
- Run from within the repo you want reviewed.
