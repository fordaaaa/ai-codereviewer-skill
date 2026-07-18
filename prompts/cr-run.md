# cr-run — parallel code review prompt

Portable version of Claude Code's `/cr-run` command. Paste this into any coding agent (Cursor, Aider, Codex, ChatGPT, etc.) — replace `{{LEVEL}}` with `low`, `medium`, or `high` first.

---

Run a code review of this repository using parallel read-only subagents (or, if your tool has no subagent/multi-session concept, simulate it by doing sequential passes with a different focus each time), then report findings ranked by severity. Do not edit any files — this is a review-only pass.

## Step 0 — parse complexity level

Level: `{{LEVEL}}` (default `medium` if you don't set one). This controls how many passes/subagents you use and how deep each goes:

- **low** — 1 pass, quick scan of the whole diff/repo for obvious correctness bugs only.
- **medium** — 3 passes/subagents, each with a focused lens: (1) correctness/logic bugs, (2) performance/efficiency/dead code, (3) security/error-handling.
- **high** — 5+ passes/subagents, split by both lens (correctness, performance, security, style/design, test coverage) AND by area of the codebase if it's large (e.g. one pass per top-level module/directory) so no single pass has to cover everything.

Scope: if there's a substantial uncommitted diff, or a PR/branch is specified, review that diff. Otherwise review the full source tree — ask if genuinely ambiguous.

## Step 1 — review

For each lens/area, read the relevant files and identify only real, verified problems (file:line, what's wrong, why it's a real issue — no speculation or unsolicited style nitpicks). Assign each finding a severity:

| Emoji | Level | Meaning |
|---|---|---|
| 🔴 | Critical (5) | crashes, data loss, security vulnerability, broken core functionality |
| 🟠 | High (4) | real bug with clear user-facing impact |
| 🟡 | Medium (3) | logic error, meaningful perf issue, or maintainability hazard in an edge case |
| 🟢 | Low (2) | minor inefficiency, dead code, unclear error handling |
| ⚪ | Trivial (1) | style/naming/cleanup, no functional impact |

## Step 2 — aggregate and dedupe

Merge all findings into one list, sorted by severity descending (🔴 → ⚪). Deduplicate anything flagged more than once. Drop anything you can't verify by rereading the cited file:line.

## Step 3 — report

Present findings as a table: severity emoji, file:line, one-line summary, one-line fix suggestion. Then ask whether to file these as GitHub issues — do not create issues without explicit confirmation.

## Step 4 — file issues (only after confirmation)

If you (or the human) have `gh` CLI access: run `gh issue list --state all --limit 100` first, skip anything already filed (compare by file/line/description, not title wording), then for each confirmed finding run `gh issue create` with:

- Title: `<severity emoji> <short description> (<file>:<line>)`
- Body: file:line, what's wrong, concrete fix suggestion, spelled-out severity level.

If you don't have `gh` access, just output the issues as formatted markdown the human can paste in manually.

Report back what was filed (or drafted) and what was left out (too trivial, needs discussion, etc).
