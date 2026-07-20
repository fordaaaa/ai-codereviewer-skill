# cr-fix — fix + PR prompt

Portable version of Claude Code's `/cr-fix` command. Paste this into any coding agent — replace `{{ISSUES}}` with an issue number, list of numbers, or `all` first.

---

Fix the following GitHub issue(s), typically ones filed by a `cr-run` review, and open a pull request. This prompt DOES involve editing code.

If your tool has an MCP server connected for an issue tracker other than GitHub (e.g. Linear), use that instead of `gh issue` — note that `Fixes #<n>`-style auto-linking is GitHub-specific, so for other trackers, reference the issue ID in the PR body and update its status manually once merged. See the Claude Code version's [MCP integrations](../README.md#mcp-integrations) for the pattern this is based on.

## Step 0 — resolve target issues

Target: `{{ISSUES}}`.

- If it's one or more issue numbers, fetch each with `gh issue view <n>` (or ask the human to paste the issue content if you have no `gh`/repo-hosting access).
- If it's `all`, list open issues first (`gh issue list --state open --limit 100`) and confirm the subset to fix before touching anything — don't silently batch-fix unrelated issues into one PR.
- If empty, ask which issue(s) to fix.

Group genuinely related issues (same file/root cause) into one PR; keep unrelated issues on separate branches/PRs.

## Step 1 — plan

Read the referenced file:line and surrounding context for each issue. Confirm it's still valid — code may have changed since filing. If stale or already fixed, say so and skip rather than forcing a change.

## Step 2 — branch

Fix everything in this run on a single branch off the default branch — don't create a branch per issue or per subagent, and don't spin up an intermediate integration branch.

- Create one working branch off the latest default branch, e.g. `git checkout -b codereview-fixes main` (name doesn't matter, pick anything reasonable).
- All issue groups from this run land as commits on that same branch. Never commit directly to the default branch.

## Step 3 — fix

Make the minimal correct change per the issue's guidance. Don't scope-creep into unrelated cleanup. Add/run tests if the fix is non-trivial and tests exist or are warranted. If multiple issues are being fixed, make one commit per issue (or per genuinely related group) on the shared branch so history stays legible, even though they all ship in one PR.

## Step 4 — verify

Actually exercise the change (run tests, run the affected code path) rather than just eyeballing the diff.

## Step 5 — commit and PR

Commit each fix with a message explaining why, referencing `Fixes #<n>` so the host (GitHub/GitLab/etc.) auto-links/closes the issue on merge. Once all targeted issues are fixed on the branch, push it and open a single PR/MR targeting the default branch. The PR title can be anything reasonable; the body must list every issue resolved (`Fixes #<n>` for each) and a short test plan covering all of them. Confirm with the human before pushing/opening the PR if that hasn't already been authorized.

Note: the `Fixes #<n>` keyword only closes the issue when the PR is *merged*, not when it's opened. Right after opening the PR, comment on each issue with the PR link so each is visibly tracked in the meantime.

## Step 6 — close out

Ask whether to merge the PR into the default branch now or leave it for review/CI first.

- Merge now: merge the PR (squash or whatever the repo convention is). This closes every issue referenced by `Fixes #<n>` in the PR body, since it lands directly on the default branch.
- Only close an issue directly (without any PR) if the fix landed via a direct commit the human asked for, or they explicitly ask you to close it out of band.

## Step 7 — report

Report the PR link, the resulting issue state (open vs. merged-and-closed) for each issue in the batch, and note any issues skipped (stale, already fixed, needs discussion).
