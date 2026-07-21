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

## Step 0.4 — load map & learned notes (optional grounding)

Cheaply load persistent context before spawning subagents so a long run doesn't re-explore from scratch:

- **Codebase map** — if `.claude/cr/codebase-map.md` exists, read it and pass the relevant sections (Modules, Entry points, Build/run) into subagent prompts so each knows the layout without re-deriving it. If missing, suggest `/cr-map` but proceed.
- **Learned notes** — if `.claude/cr/learn.enabled` exists, read `.claude/cr/notes.md` and fold it in: honor `false-positive` notes, give `hotspot` areas extra scrutiny, respect `convention` notes so you don't flag intentional patterns. If learning is off, skip silently.

## Step 0.5 — static analysis grounding (optional)

Before spawning subagents, check `ToolSearch` (query `"mcp__semgrep"`) for a configured Semgrep MCP server. If available, run it once over the review scope using its correctness/best-practice rulesets (not just security) and pass raw hits into the relevant subagent's prompt in Step 1 as supporting evidence — a subagent must independently verify any tool hit against the actual code before reporting it as a finding, since Semgrep has its own false positives. If unavailable, skip this step and proceed exactly as before.

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
- Assign each finding a confidence score 1-10; only report findings scoring 8+ (>80% confident it's a real, reproducible issue) — drop the rest rather than including them as caveats.
- Apply these hard exclusions (skip unless `high` tier, where they may be noted briefly as low-priority): style/naming nitpicks with no functional impact unless explicitly requested; purely theoretical issues with no concrete triggering input/state; findings in test-only files unless the bug is in the test's own logic (e.g. an assertion that can't fail).
- Cap response length (e.g. "under 300 words, bullet list") to keep aggregation cheap.

## Step 2 — aggregate and dedupe

Once subagents report back, merge all findings into one list, sorted by severity descending (🔴 → ⚪). Deduplicate anything two subagents flagged independently. Drop anything you can't personally verify by spot-checking the cited file:line yourself. Mark any finding also flagged by Semgrep (Step 0.5) as tool-confirmed — call this out distinctly in the Step 3 report.

## Step 3 — report to the user

Present the findings as a table or list: severity emoji, file:line, one-line summary, one-line fix suggestion. Then ask the user explicitly whether to file GitHub issues for some/all of these findings — do not create issues without this confirmation.

## Step 4 — file issues (only after user confirms)

Pick a tracker per [Tracker selection](#tracker-selection) below. List existing open items first and skip anything already filed (compare by file/line/description, not title wording). For each confirmed finding, create an item with:

- Title: `<severity emoji> <short description> (<file>:<line>)`
- Body: file:line, what's wrong, concrete fix suggestion, severity level (spelled out), and a severity label/tag if the tracker supports one (create new labels only if the user agrees).

Report back the filed item links, and list anything left unfiled (too trivial, needs more discussion, etc).

## Step 5 — record lessons (only if learning is enabled)

If `.claude/cr/learn.enabled` exists (see `/cr-learn`), append any genuinely new, durable lessons to `.claude/cr/notes.md`, deduping against existing notes — e.g. a `hotspot` note for an area with a confirmed bug, a `convention` note for an intentional pattern you initially misread, or a `false-positive` note for something you confirmed is fine. Terse and durable only; skip if learning is off.

## Tracker selection

Before filing anything, check `ToolSearch` (query `"mcp__linear"`) for a configured Linear MCP server:

- **If Linear MCP tools are available**, use them (`create_issue`, `list_issues`, `update_issue`, `create_comment`, etc.) instead of `gh issue`. Ask the user which Linear team/project to file into if it's not obvious from context.
- **Otherwise**, use the `github` MCP server's tools if configured, or plain `gh issue` (GitHub CLI) if not — either way, GitHub issues as before.

See the root [README](../../README.md#mcp-integrations) for how to configure the Linear MCP server.
