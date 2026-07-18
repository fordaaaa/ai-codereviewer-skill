# cr-fix — fix + PR prompt

Portable version of Claude Code's `/cr-fix` command. Paste this into any coding agent — replace `{{ISSUES}}` with an issue number, list of numbers, or `all` first.

---

Fix the following GitHub issue(s), typically ones filed by a `cr-run` review, and open a pull request. This prompt DOES involve editing code.

## Step 0 — resolve target issues

Target: `{{ISSUES}}`.

- If it's one or more issue numbers, fetch each with `gh issue view <n>` (or ask the human to paste the issue content if you have no `gh`/repo-hosting access).
- If it's `all`, list open issues first (`gh issue list --state open --limit 100`) and confirm the subset to fix before touching anything — don't silently batch-fix unrelated issues into one PR.
- If empty, ask which issue(s) to fix.

Group genuinely related issues (same file/root cause) into one PR; keep unrelated issues on separate branches/PRs.

## Step 1 — plan

Read the referenced file:line and surrounding context for each issue. Confirm it's still valid — code may have changed since filing. If stale or already fixed, say so and skip rather than forcing a change.

## Step 2 — branch

Create a new branch per PR-worthy group, e.g. `fix/<short-slug>`. Never commit directly to the main/default branch.

## Step 3 — fix

Make the minimal correct change per the issue's guidance. Don't scope-creep into unrelated cleanup. Add/run tests if the fix is non-trivial and tests exist or are warranted.

## Step 4 — verify

Actually exercise the change (run tests, run the affected code path) rather than just eyeballing the diff.

## Step 5 — commit and PR

Commit with a message explaining why, referencing `Fixes #<n>` so the host (GitHub/GitLab/etc.) auto-links/closes the issue on merge. Push the branch and open a PR/MR with a body listing which issue(s) it fixes and a short test plan. Confirm with the human before pushing/opening the PR if that hasn't already been authorized.

## Step 6 — report

Report the PR link(s) and note any issues skipped (stale, already fixed, needs discussion).
