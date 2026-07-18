---
description: Run a parallel multi-subagent code review of the repo and report findings by severity (optionally filing GitHub issues)
argument-hint: "[low|medium|high]"
---

Run a code review of this repository using parallel read-only subagents, then report findings ranked by severity. This command never edits files by itself — see `/cr-fix` for that.

## Step 0 — parse complexity level

Argument: `$ARGUMENTS` (default `medium` if empty). This controls how many subagents you spawn and how deep each one goes:

- **low** — 1 subagent, single quick pass over the whole diff/repo for obvious correctness bugs only.
- **medium** — 3 subagents in parallel, each with a focused lens: (1) correctness/logic bugs, (2) performance/efficiency/dead code, (3) security/error-handling. Each subagent reads the relevant source files itself.
- **high** — 5+ subagents in parallel: split by both lens (correctness, performance, security, style/design, test coverage) AND by area of the codebase if it's large (e.g. one subagent per top-level module/directory) so no single subagent has to read everything.

Scope: if the repo has a substantial uncommitted diff or the user references a PR/branch, review that diff. Otherwise review the full source tree (use judgment — ask the user if genuinely ambiguous, e.g. "review everything or just recent changes?").

## Step 1 — spawn subagents

Use the `Agent` tool with `subagent_type: Explore` or `general-purpose` (read-only findings, no edits) for each lens/area determined above, in parallel (single message, multiple tool calls). Each subagent prompt must:

- State exactly which files/directories or diff it's responsible for.
- Ask it to return ONLY verified, concrete findings (file:line, what's wrong, why it's a real bug/issue — not speculation or style nitpicks unless explicitly asked).
- Ask it to self-assign a severity per finding using this scale:
  - 🔴 Critical (5) — crashes, data loss, security vulnerability, broken core functionality
  - 🟠 High (4) — real bug with clear user-facing impact, but not catastrophic
  - 🟡 Medium (3) — logic error, meaningful perf issue, or maintainability hazard in an edge case
  - 🟢 Low (2) — minor inefficiency, dead code, unclear error handling
  - ⚪ Trivial (1) — style/naming/cleanup, no functional impact
- Cap response length (e.g. "under 300 words, bullet list") to keep aggregation cheap.

## Step 2 — aggregate and dedupe

Once subagents report back, merge all findings into one list, sorted by severity descending (🔴 → ⚪). Deduplicate anything two subagents flagged independently. Drop anything you can't personally verify by spot-checking the cited file:line yourself.

## Step 3 — report to the user

Present the findings as a table or list: severity emoji, file:line, one-line summary, one-line fix suggestion. Then ask the user explicitly whether to file GitHub issues for some/all of these findings — do not create issues without this confirmation.

## Step 4 — file issues (only after user confirms)

Run `gh issue list --state all --limit 100` first and skip anything already filed (compare by file/line/description, not title wording). For each confirmed finding, run `gh issue create` with:

- Title: `<severity emoji> <short description> (<file>:<line>)`
- Body: file:line, what's wrong, concrete fix suggestion, severity level (spelled out), and label via `--label` if the repo has matching severity labels (create them with `gh label create` only if the user agrees to that too).

Report back the filed issue links, and list anything left unfiled (too trivial, needs more discussion, etc).
