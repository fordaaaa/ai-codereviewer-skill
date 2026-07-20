# cr-recheck — stale-issue triage prompt

Portable version of Claude Code's `/cr-recheck` command. Paste this into any coding agent — replace `{{ISSUES}}` with an issue number, list of numbers, or `all` first.

Run this before `cr-fix.md` on a batch of old issues, so you don't spend fix effort re-investigating something that already got fixed, refactored away, or deleted while other work continued in the meantime. Read-only with respect to code — only reads source, never edits it; only updates/closes issues.

---

## Step 0 — resolve target issues

Target: `{{ISSUES}}` (default: all open issues previously filed by a `cr-run`/`cr-sec` style review).

- If one or more issue numbers, fetch each with `gh issue view <n>` (or ask the human to paste the issue content if you have no `gh`/repo-hosting access).
- If `all`, list open issues first (`gh issue list --state open --limit 100`) and use every one that looks like a review finding (has a `file:line` reference, or a severity/category label).
- If there are none, say so and stop.

## Step 1 — refresh each issue's location

For each issue, don't trust the recorded file:line as gospel — the code may have moved:

1. Read the file at the recorded line plus the enclosing function/block for context.
2. If the described problem is clearly still there at that location: **confirmed**.
3. If the file/function still exists but the line shifted (reformatting, code added above it), search the file for the original snippet/symbol named in the issue body and re-locate it. If found and still applies: **confirmed**, note the corrected line.
4. If the code at that location no longer matches the issue's description — already fixed, refactored, or logic changed such that it no longer applies: **stale**.
5. If the file or function was deleted entirely: **stale**.
6. If you genuinely can't tell from reading the code alone: **needs-human**.

If your tool supports parallel sub-tasks, run one per issue (or small batch of issues in the same file) to keep this cheap; otherwise do sequential passes.

## Step 2 — act on results

- **confirmed**: leave open. If the line moved, comment with the corrected `file:line`.
- **stale**: comment explaining specifically why it no longer applies, then close it (`gh issue close <n> --comment "..."` if you have `gh` access, otherwise tell the human which issues are safe to close and why). Never delete — close with the explanation as history.
- **needs-human**: comment noting what's ambiguous, leave open, flag for the human.

## Step 3 — report

Give a summary table: issue number, verdict, one-line reason, and how many were closed as stale before any fix effort was spent on them.
