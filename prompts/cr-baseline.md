# cr-baseline — findings baseline prompt

Portable version of Claude Code's `/cr-baseline` command. Paste into any coding agent. Replace `{{ACTION}}` with one of: `list`, `accept <file:line — desc>`, `remove <id>`, `clear`, `snapshot` (default `list`).

---

Manage a **findings baseline** — issues you've already triaged and decided not to act on — so review passes report only *new* problems. Baselined findings are suppressed from the main report and shown as a single "N suppressed" count. This differs from a learning-notes store: notes explain *why* something is safe; the baseline is a blunt "don't show me this again" list.

Baseline lives at `.claude/cr/baseline.md`.

Action: `{{ACTION}}`
- `list` — print entries grouped by file.
- `accept <file:line — description>` — add one entry.
- `remove <id>` — drop an entry so it surfaces again.
- `clear` — empty the baseline (confirm first).
- `snapshot` — add all findings from the most recent review at once (clean-slate a legacy codebase). Confirm count first.

## Schema

`.claude/cr/baseline.md` starts with:

```
# Findings Baseline
<!-- Suppressed from review reports. Managed by cr-baseline. -->
```

Each entry:

```
### B<id> · <file:line> · <category> · accepted <ISO-date>
<one-line description> — reason: <accepted-risk|wont-fix|env-specific|other>
```

## Matching

Suppress a live finding if it matches an entry by file + category + fuzzy description, tolerant of line drift — match on *what* it is, not the exact line. If the code region changed in a way that could affect the issue, do NOT suppress; surface it and flag the baseline entry as possibly stale.

## In review passes

Read `.claude/cr/baseline.md` if present, drop matching findings from the main report, and replace them with one suppressed-count line. New findings unaffected. Baselining is not fixing — remove an entry when ready to deal with it.
