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

## Step 3 — report to the user

Present findings as a table: severity, category, file:line, one-line summary, exploit scenario, fix suggestion. Then ask the user explicitly whether to file GitHub issues for some/all of these — do not create issues without confirmation.

## Step 4 — file issues (only after user confirms)

Run `gh issue list --state all --limit 100` first and skip anything already filed (compare by file/line/description, not title wording). For each confirmed finding, run `gh issue create` with:

- Title: `[security] <category> — <short description> (<file>:<line>)`
- Body: file:line, description, exploit scenario, fix recommendation, severity, and a `security` label (plus severity label if the repo has matching labels — create with `gh label create` only if the user agrees).

Report back the filed issue links, and list anything left unfiled (too speculative, needs more discussion, etc). Findings filed this way are compatible with `/cr-fix` for remediation.
