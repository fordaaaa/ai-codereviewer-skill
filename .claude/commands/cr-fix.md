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

Fix everything in this run on a single branch off `main` — don't create a branch per issue or per subagent, and don't spin up an intermediate integration branch.

- Create one working branch off the latest `main`, e.g. `git checkout -b codereview-fixes main` (name doesn't matter, pick anything reasonable).
- All issue groups from this run land as commits on that same branch. Never commit directly to `main`.

## Step 3 — fix

Make the minimal correct change per the guidance already in the issue body. Don't scope-creep into unrelated cleanup. If tests exist, run them; if not, and the fix is non-trivial, consider whether a test is warranted (ask the user if unsure). If multiple issues are being fixed, make one commit per issue (or per genuinely related group) on the shared branch so history stays legible, even though they all ship in one PR.

## Step 4 — verify

Actually exercise the change if there's a way to (run tests, run the affected code path) — don't just eyeball the diff. Use the `verify` skill if applicable.

## Step 5 — commit and PR

Commit each fix with a message describing why (referencing `Fixes #<n>` so GitHub auto-links/closes the issue on merge). Once all targeted issues are fixed on the branch, push it and open a single PR with `gh pr create --base main`. The PR title can be anything reasonable (e.g. summarizing the theme, or just "Code review fixes"); the PR body must list every issue resolved (`Fixes #<n>` for each) and a short test plan covering all of them. Confirm with the user before pushing/opening the PR if they haven't already authorized this flow.

Note: `Fixes #<n>` only auto-closes an issue when the PR is *merged* — opening the PR alone does not close it. Comment on each issue with the PR link right after opening it (`gh issue comment <n> --body "PR: <url>"`) so each is visibly tracked as in-progress.

## Step 6 — close out

Ask the user whether to merge the PR into `main` now or leave it for review/CI first.

- If they say merge now: run `gh pr merge --squash` (or whatever the repo's convention is). This closes every issue referenced by `Fixes #<n>` in the PR body, since it lands directly on `main`.
- Never close an issue manually with `gh issue close` unless the fix landed without a PR (e.g. direct commit the user asked for) or the user explicitly asks you to close it out of band.

## Step 7 — report

Give the user the PR link, current issue state (open/merged-and-closed) for each issue in the batch, and note any issues you skipped (stale, already fixed, needs discussion).
