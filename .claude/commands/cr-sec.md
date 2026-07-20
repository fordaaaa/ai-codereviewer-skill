---
description: Run a parallel multi-subagent SECURITY review of the repo and report vulnerabilities by severity (optionally filing GitHub issues)
argument-hint: "[low|medium|high]"
---

Run a security-focused review of this repository using parallel read-only subagents, then report vulnerability findings ranked by severity. This is the security counterpart to `/cr-run` — use `/cr-run` for general correctness/perf/style review, `/cr-sec` for security only. This command never edits files by itself — see `/cr-fix` for that.

The vulnerability taxonomy, exclusion list, and severity/confidence scoring below are adapted from Anthropic's open-source [`claude-code-security-review`](https://github.com/anthropics/claude-code-security-review) action (MIT licensed) — see the README for attribution.

## Step 0 — parse complexity level

Argument: `$ARGUMENTS` (default `medium` if empty).

- **low** — 1 subagent, single quick pass over the diff/repo for high-confidence, high-impact vulnerabilities only.
- **medium** — 3 subagents in parallel, split by category group: (1) input validation & injection (SQLi, command injection, XXE, template/NoSQL injection, path traversal), (2) auth/crypto/secrets (authn/authz bypass, session/JWT flaws, hardcoded secrets, weak crypto), (3) code execution & data exposure (insecure deserialization, eval injection, XSS, sensitive data logging/leakage).
- **high** — 5+ subagents: the same category split AND by area of the codebase if it's large (one subagent per top-level module/directory) so no single subagent has to read everything.

Scope: if there's a substantial uncommitted diff, or a PR/branch is specified, review only that diff (this is the common case — security review is usually run per-PR). Otherwise review the full source tree — ask if genuinely ambiguous.

## Step 0.5 — static analysis grounding (optional)

Before spawning subagents, check `ToolSearch` (query `"mcp__semgrep"`) for a configured Semgrep MCP server:

- **If available**, run it once over the review scope (the diff's files, or the full tree) using its default/security rulesets. Collect the raw findings (rule id, file:line, message).
- **If a `gitleaks` binary is available** (check with a quick `which gitleaks`/`gitleaks version`), run `gitleaks detect` (or `gitleaks protect` for a diff) over the same scope for hardcoded secrets.
- Pass any raw hits into the relevant subagent's prompt in Step 1 as supporting evidence, tagged by category (Semgrep secrets/crypto hits and gitleaks hits go to the auth/crypto/secrets subagent; other Semgrep rule categories go to the matching subagent). Subagents must independently verify each tool hit against the actual code — a Semgrep/gitleaks match is a lead, not an automatic finding, since both tools have their own false positives.
- **If neither tool is available**, skip this step and proceed exactly as before — note in the final report that findings are LLM-only review (no static-analysis grounding) so the user knows the confidence bar reflects that.

## Step 1 — spawn subagents

Use the `Agent` tool with `subagent_type: Explore` or `general-purpose` (read-only, no edits) for each category/area, in parallel (single message, multiple tool calls). Each subagent prompt must include:

**Objective**: Identify HIGH-CONFIDENCE security vulnerabilities with real exploitation potential — newly introduced or pre-existing in its assigned scope. Not a general code review; skip style/theoretical/low-impact issues.

**Categories to examine** (assign the relevant subset per subagent):
- Input validation: SQL injection, command injection, XXE, template injection, NoSQL injection, path traversal
- Auth & authz: authentication bypass, privilege escalation, session management flaws, JWT vulnerabilities, authorization bypasses
- Crypto & secrets: hardcoded API keys/passwords/tokens, weak crypto algorithms, improper key storage, weak randomness, certificate validation bypasses
- Injection & code execution: insecure deserialization (pickle, YAML), eval injection, RCE, XSS (reflected/stored/DOM)
- Data exposure: sensitive data logging, PII handling violations, API data leakage, debug info exposure

**Hard exclusions** (do not report these):
- Denial-of-service / resource exhaustion / rate limiting
- Secrets on disk that are otherwise secured
- Outdated third-party library vulnerabilities (handled separately, e.g. dependabot)
- Memory safety issues (buffer overflow, use-after-free) in memory-safe languages
- Findings in test-only files or documentation/markdown files
- Log spoofing from unsanitized input in logs; logging URLs
- SSRF that only controls the path (not host/protocol)
- Regex injection / regex DoS
- Lack of general hardening/best-practices with no concrete exploit
- Race conditions that are theoretical rather than concretely problematic
- Client-side JS/TS lacking permission checks (server is the trust boundary)
- React/Angular XSS unless using `dangerouslySetInnerHTML`/`bypassSecurityTrustHtml` or similar unsafe escapes
- Command injection in shell scripts unless there's a concrete untrusted-input path
- Environment variables / CLI flags treated as attacker-controlled

**Severity scale**: HIGH (RCE, data breach, auth bypass), MEDIUM (significant impact but needs specific conditions), LOW (defense-in-depth). Only report LOW if grouped as minor notes — focus output on HIGH/MEDIUM.

**Confidence**: assign 1-10; only include findings scoring 8+ (i.e. >80% confident of actual exploitability). Drop anything below that threshold rather than including it as a caveat.

**Output per finding**: file:line, severity, category (e.g. `sql_injection`, `xss`), description, concrete exploit scenario, fix recommendation. Cap response length (e.g. under 300 words per finding, bullet list).

## Step 2 — aggregate, dedupe, and false-positive filter

Once subagents report back:

1. Merge all findings, sorted by severity descending (HIGH → LOW).
2. Deduplicate anything two subagents flagged independently.
3. For each remaining finding, spot-check the cited file:line yourself against the hard-exclusions list above and drop any that match — err toward dropping rather than flooding the report with noise.
4. Drop anything you can't personally verify by reading the cited code.
5. Mark any finding that was also flagged by Semgrep or gitleaks (Step 0.5) as **tool-confirmed** in the report — this is a stronger signal than LLM-only findings and should be called out distinctly (e.g. a column/tag) in Step 3.
6. If `WebSearch`/`WebFetch` are available and a finding involves a specific third-party library/version, do a quick search for a matching known CVE or advisory (e.g. NVD, GitHub Advisory Database) and cite it if found — this turns a vague "this pattern looks risky" into a concrete "this matches CVE-2024-XXXX." Don't block the report waiting on this; skip it if the lookup is inconclusive.
7. If a `postgres` MCP server is configured and a finding involves a database query, use it to check the actual schema/column types instead of guessing from the code — this sharpens SQL-injection findings (e.g. confirming a raw string really does reach a query, not just something that looks like one) and helps rule out false positives where an ORM already parameterizes the call.

If a Sentry MCP server is configured (see [Tracker selection](#tracker-selection)), cross-reference each HIGH finding against recent production errors/issues in Sentry before finalizing severity — a finding that's already causing crashes in prod should be called out explicitly as such.

## Step 3 — report to the user

Present findings as a table: severity, category, file:line, one-line summary, exploit scenario, fix suggestion. Then ask the user explicitly whether to file issues for some/all of these — do not create issues without confirmation.

If a Slack MCP server is configured and the user wants findings posted there too, ask which channel, then post a short summary of HIGH-severity findings only (channel messages should stay skimmable — link to the filed issues rather than repeating full detail).

## Step 4 — file issues (only after user confirms)

Pick a tracker per [Tracker selection](#tracker-selection) below. List existing open items first and skip anything already filed (compare by file/line/description, not title wording). For each confirmed finding, create an item with:

- Title: `[security] <category> — <short description> (<file>:<line>)`
- Body: file:line, description, exploit scenario, fix recommendation, severity, and a `security` label/tag (plus severity label if the tracker has matching labels — create with `gh label create` or the tracker's equivalent only if the user agrees).

Report back the filed item links, and list anything left unfiled (too speculative, needs more discussion, etc). Findings filed this way are compatible with `/cr-fix` for remediation.

## Tracker selection

Before filing anything, check `ToolSearch` (query `"mcp__linear"`) for a configured Linear MCP server:

- **If Linear MCP tools are available**, use them (`create_issue`, `list_issues`, `update_issue`, `create_comment`) instead of `gh issue`. Ask the user which Linear team/project to file into if it's not obvious.
- **Otherwise**, use the `github` MCP server's tools if configured, or plain `gh issue` (GitHub CLI) if not — either way, GitHub issues as before.

See the root [README](../../README.md#mcp-integrations) for how to configure Linear, Slack, and Sentry MCP servers.
