---
description: Toggle and manage a persistent per-repo notes store that lets Claude record and reuse what it learns across cr-* runs (off by default)
argument-hint: "[on|off|status|note <text>|show|forget <id>]"
---

Manage a durable **learning store** for this repo so Claude stops re-deriving the same facts on every long run. When enabled, the `cr-*` commands read accumulated notes at the start of a run and append new durable lessons at the end. **This is off by default and only does anything once explicitly turned on** — it changes what future runs read, so the user opts in deliberately.

State lives in **`.claude/cr/`**:
- `.claude/cr/learn.enabled` — a marker file; its presence means learning is ON.
- `.claude/cr/notes.md` — the append-only notes store (structured, see schema below).

## Step 0 — parse argument

Argument: `$ARGUMENTS` (default: `status`).

- `on` — create `.claude/cr/learn.enabled` (write the enabling date inside it). Create `.claude/cr/notes.md` with the header below if missing. Confirm it's on.
- `off` — delete `.claude/cr/learn.enabled`. **Do not delete `notes.md`** — notes are kept so re-enabling resumes where it left off. Confirm it's off and that notes are preserved.
- `status` — report whether learning is on, how many notes are stored, and the last-updated date.
- `note <text>` — manually append a note the user dictates (same schema as auto-notes, `source: user`). Allowed even while learning is off.
- `show` — print the current notes grouped by category.
- `forget <id>` — remove the note with that id (or mark it `retracted` if it's referenced elsewhere), e.g. when a note has gone stale.

## Notes schema

`.claude/cr/notes.md` starts with:

```
# Learned Notes
<!-- Written by cr-* runs when learning is enabled (`/cr-learn on`). Terse, durable facts only. -->
```

Each note is one block. Keep them short and durable — a note must still be true next month:

```
### N<id> · <category> · <ISO-date> · source: auto|user
<one or two lines of the actual lesson> — <file:path or area if code-specific>
```

Categories: `convention` (project style/patterns to respect), `false-positive` (a finding class already confirmed safe here — don't re-flag), `hotspot` (an area that has repeatedly had real bugs), `arch` (a non-obvious design fact), `gotcha` (a trap that wasted time before).

## What counts as a good note (and what doesn't)

Record only things that (a) were **non-obvious**, (b) will **recur**, and (c) **aren't already in the code, git history, or the codebase map**. Good: "raw SQL in `reports/export.ts` is safe — the string is a fixed allowlist, not user input; stop flagging it." Bad: restating what a function does, one-off task state, or anything a future run could trivially re-read. When in doubt, don't write it — a noisy notes file is worse than none.

## Behavior in other commands

When `.claude/cr/learn.enabled` exists, `/cr-run` and `/cr-sec` will, on their own:
1. **Read `notes.md` at the start** and fold it into subagent prompts (respect `false-positive` notes to cut noise; give `hotspot` areas extra scrutiny).
2. **Append at the end** any genuinely new, durable lessons discovered during the run (auto notes), deduping against existing ones.

If learning is off, those commands neither read nor write the store. This command is the only switch.

## Step — execute and report

Perform the parsed action, then report the resulting state (on/off, note count, path to `notes.md`). Remind the user that `.claude/cr/notes.md` is committed by default so the whole team benefits — if they'd rather keep it local, add `.claude/cr/` to `.gitignore`.
