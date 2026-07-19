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

All fixes go through a shared `codereview` integration branch instead of branching straight off the default branch:

- If `codereview` already exists (local or remote), check it out and bring it up to date with the default branch (fast-forward if possible; confirm before force-updating if it's diverged).
- If it doesn't exist, create it from the current default branch: `git checkout -b codereview main` (adjust `main` if the default branch has another name).
- Branch each PR-worthy group off `codereview`, e.g. `fix/<short-slug>`, and target the eventual PR/MR at `codereview`, not the default branch.

Never commit directly to the default branch or to `codereview` itself — always work on a `fix/<short-slug>` branch on top of it.

## Step 3 — fix

Make the minimal correct change per the issue's guidance. Don't scope-creep into unrelated cleanup. Add/run tests if the fix is non-trivial and tests exist or are warranted.

## Step 4 — verify

Actually exercise the change (run tests, run the affected code path) rather than just eyeballing the diff.

## Step 5 — commit and PR

Commit with a message explaining why, referencing `Fixes #<n>` so the host (GitHub/GitLab/etc.) auto-links/closes the issue on merge. Push the branch and open a PR/MR targeting `codereview` with a body listing which issue(s) it fixes and a short test plan. Confirm with the human before pushing/opening the PR if that hasn't already been authorized.

Note: the `Fixes #<n>` keyword only closes the issue when the PR is *merged*, not when it's opened. Right after opening the PR, comment on the issue with the PR link so it's visibly tracked in the meantime.

## Step 6 — close out

Ask whether to merge the `fix/<slug>` PR into `codereview` now or leave it for review/CI first.

- Merge now: merge the PR into `codereview` (squash or whatever the repo convention is).
- **Important:** `Fixes #<n>` only auto-closes an issue when it lands on the repo's *default* branch, so merging into `codereview` alone will NOT close the issue — it just accumulates fixes there. Say this explicitly.
- The issue actually closes once `codereview` is merged into the default branch (a separate step — only do this when asked, e.g. once several fixes have been batched up and are ready to ship).
- Only close an issue directly (without any PR) if the fix landed via a direct commit the human asked for, or they explicitly ask you to close it out of band.

## Step 7 — report

Report the PR link(s), the resulting issue state (open vs. merged-and-closed), and note any issues skipped (stale, already fixed, needs discussion).
