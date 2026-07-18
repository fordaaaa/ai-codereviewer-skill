---
description: Fix one or more GitHub issues filed by /cr-run and open a pull request
argument-hint: "<issue-number> [issue-number ...] | all"
---

Fix GitHub issue(s) previously filed (typically by `/cr-run`) and open a pull request. This command DOES edit code, unlike `/cr-run`.

## Step 0 — resolve target issues

Argument: `$ARGUMENTS`.

- If it's one or more issue numbers, fetch each with `gh issue view <n>`.
- If it's `all`, run `gh issue list --state open --limit 100` and ask the user to confirm the subset to fix before touching anything (don't silently fix everything — batch-fixing unrelated issues in one PR is usually wrong).
- If empty, ask the user which issue(s) to fix.

Group issues that are genuinely related (same file/root cause) into one PR; keep unrelated issues on separate branches/PRs unless the user says otherwise.

## Step 1 — plan

For each issue (or group), read the referenced file:line and surrounding context. Confirm the issue is still valid (code may have changed since filing) — if it's stale or already fixed, say so and skip it rather than forcing a change.

## Step 2 — branch

Create a new branch per PR-worthy group, e.g. `fix/<short-slug>`. Never commit directly to main.

## Step 3 — fix

Make the minimal correct change per the guidance already in the issue body. Don't scope-creep into unrelated cleanup. If tests exist, run them; if not, and the fix is non-trivial, consider whether a test is warranted (ask the user if unsure).

## Step 4 — verify

Actually exercise the change if there's a way to (run tests, run the affected code path) — don't just eyeball the diff. Use the `verify` skill if applicable.

## Step 5 — commit and PR

Commit with a message describing why (referencing `Fixes #<n>` so GitHub auto-links/closes the issue on merge). Push the branch and open a PR with `gh pr create`, body listing which issue(s) it fixes and a short test plan. Confirm with the user before pushing/opening the PR if they haven't already authorized this flow.

## Step 6 — report

Give the user the PR link(s) and note any issues you skipped (stale, already fixed, needs discussion).
