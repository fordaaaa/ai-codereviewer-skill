---
description: Manage a baseline of accepted/known findings so cr-run and cr-sec report only NEW issues, not ones you've already triaged
argument-hint: "[list|accept <file:line — desc>|remove <id>|clear|snapshot]"
---

Manage a **findings baseline** — the set of issues you've already looked at and decided not to act on (accepted risk, won't-fix, false positive that isn't worth a permanent `false-positive` note). Once baselined, `/cr-run` and `/cr-sec` suppress these from their main report and instead show a one-line "N baselined findings suppressed" count, so every review focuses on what's *new*.

The baseline lives at **`.claude/cr/baseline.md`** (gitignored in this repo; consumers commit their own). It's distinct from `/cr-learn` notes: notes teach Claude *why* something is safe; the baseline is a blunt "don't show me this again" list keyed by location + fingerprint.

## Step 0 — parse argument

Argument: `$ARGUMENTS` (default: `list`).

- `list` — print current baseline entries grouped by file.
- `accept <file:line — description>` — add one entry manually.
- `remove <id>` — drop an entry (so it will surface again next review).
- `clear` — empty the baseline (confirm first).
- `snapshot` — run/read the most recent review findings and add all of them to the baseline at once (use after an initial review of a legacy codebase where you want a clean slate going forward). Confirm the count before writing.

## Baseline schema

`.claude/cr/baseline.md` starts with:

```
# Findings Baseline
<!-- Suppressed from cr-run/cr-sec main reports. Managed by /cr-baseline. -->
```

Each entry:

```
### B<id> · <file:line> · <category> · accepted <ISO-date>
<one-line description of the finding> — reason: <accepted-risk|wont-fix|env-specific|other>
```

## Matching (how suppression works)

A live finding is considered baselined if it matches an entry by **file + category + a fuzzy description match**, tolerant of line-number drift (the surrounding code may have shifted). Match on *what* the finding is, not the exact line — otherwise a one-line edit above it would resurrect it. If a baselined finding's code region has clearly changed in a way that could affect the issue, do NOT suppress it — surface it and note "baselined entry may be stale (code changed nearby)".

## Behavior in other commands

At Step 0.4, `/cr-run` and `/cr-sec` read `.claude/cr/baseline.md` if present and drop matching findings from the main report, replacing them with a single suppressed-count line (and a hint to run `/cr-baseline list` to review them). New findings are unaffected.

## Step — execute and report

Perform the parsed action, then report the resulting baseline size and path. Remind the user that baselining is not fixing — `/cr-baseline remove <id>` un-suppresses an entry when they're ready to deal with it.
