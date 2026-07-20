# cr-sec — parallel SECURITY review prompt

Portable version of Claude Code's `/cr-sec` command. Paste this into any coding agent (Cursor, Aider, Codex, ChatGPT, etc.) — replace `{{LEVEL}}` with `low`, `medium`, or `high` first.

This is the security-focused counterpart to `cr-run.md` — use that one for general correctness/perf/style review, this one for vulnerabilities only. Do not edit any files — this is a review-only pass.

The vulnerability taxonomy, exclusion list, and confidence scoring below are adapted from Anthropic's open-source [`claude-code-security-review`](https://github.com/anthropics/claude-code-security-review) action (MIT licensed) — see the repo README for attribution.

If your tool has an MCP server connected for an issue tracker (Linear), CVE/advisory lookup, or Slack, use those instead of `gh issue`/manual search wherever this prompt mentions them. See the Claude Code version's [MCP integrations](../README.md#mcp-integrations) for the pattern this is based on.

---

Run a security review of this repository using parallel read-only subagents (or, if your tool has no subagent/multi-session concept, simulate it with sequential passes, one category group per pass), then report vulnerability findings ranked by severity.

## Step 0 — parse complexity level

Level: `{{LEVEL}}` (default `medium` if you don't set one).

- **low** — 1 pass, quick scan for high-confidence, high-impact vulnerabilities only.
- **medium** — 3 passes, split by category group: (1) input validation & injection, (2) auth/crypto/secrets, (3) code execution & data exposure.
- **high** — 5+ passes, same category split AND by area of the codebase if it's large.

Scope: if there's a substantial uncommitted diff, or a PR/branch is specified, review that diff (the common case). Otherwise review the full source tree — ask if genuinely ambiguous.

## Step 0.5 — static analysis grounding (optional)

If your tool can run shell commands and `semgrep` and/or `gitleaks` are installed, run them over the review scope before your own read: `semgrep --config auto <path>` for static analysis, `gitleaks detect`/`gitleaks protect` for secrets. Treat their output as leads, not findings — verify each hit against the actual code yourself before reporting it, since both tools produce false positives. If neither is installed, skip this step and note in the final report that the review is LLM-only.

## Step 1 — review

**Objective**: identify HIGH-CONFIDENCE security vulnerabilities with real exploitation potential. Not a general code review — skip style/theoretical/low-impact issues.

**Categories:**
- Input validation: SQL injection, command injection, XXE, template injection, NoSQL injection, path traversal
- Auth & authz: authentication bypass, privilege escalation, session management flaws, JWT vulnerabilities, authorization bypasses
- Crypto & secrets: hardcoded API keys/passwords/tokens, weak crypto algorithms, improper key storage, weak randomness, certificate validation bypasses
- Injection & code execution: insecure deserialization (pickle, YAML), eval injection, RCE, XSS (reflected/stored/DOM)
- Data exposure: sensitive data logging, PII handling violations, API data leakage, debug info exposure

**Hard exclusions** (do not report):
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

**Severity**: HIGH (RCE, data breach, auth bypass), MEDIUM (significant impact, needs specific conditions), LOW (defense-in-depth, only note briefly).

**Confidence**: score each finding 1-10; only keep findings scoring 8+ (>80% confident of actual exploitability). Drop the rest rather than including them as caveats.

**Per finding**, capture: file:line, severity, category (e.g. `sql_injection`, `xss`), description, concrete exploit scenario, fix recommendation.

## Step 2 — aggregate, dedupe, filter

Merge findings, sorted by severity descending. Deduplicate anything flagged more than once. Re-check each remaining finding against the hard-exclusions list and drop matches — err toward dropping rather than flooding the report with noise. Drop anything you can't verify by rereading the cited code. Mark any finding also caught by Semgrep/gitleaks (Step 0.5) as tool-confirmed in the report.

## Step 3 — report

Present findings as a table: severity, category, file:line, one-line summary, exploit scenario, fix suggestion. Then ask whether to file these as GitHub issues — do not create issues without explicit confirmation.

## Step 4 — file issues (only after confirmation)

If you (or the human) have `gh` CLI access: run `gh issue list --state all --limit 100` first, skip anything already filed, then for each confirmed finding run `gh issue create` with:

- Title: `[security] <category> — <short description> (<file>:<line>)`
- Body: file:line, description, exploit scenario, fix recommendation, severity, and a `security` label if the repo has one.

If you don't have `gh` access, output the issues as formatted markdown the human can paste in manually.

Report back what was filed (or drafted) and what was left out (too speculative, needs discussion, etc). Findings filed this way are compatible with `cr-fix.md` for remediation.
