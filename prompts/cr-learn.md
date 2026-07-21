# cr-learn — persistent learning store prompt

Portable version of Claude Code's `/cr-learn` command. Paste this into any coding agent. Replace `{{ACTION}}` with one of: `on`, `off`, `status`, `note <text>`, `show`, `forget <id>` (default `status`).

---

Manage a durable **learning store** for this repo so the agent stops re-deriving the same facts on every long run. When enabled, review passes read accumulated notes at the start and append new durable lessons at the end. **Off by default** — it only affects future runs once explicitly turned on.

State lives in `.claude/cr/`:
- `.claude/cr/learn.enabled` — marker file; its presence means learning is ON.
- `.claude/cr/notes.md` — the append-only notes store.

Action: `{{ACTION}}`
- `on` — create `.claude/cr/learn.enabled` (write the enabling date inside). Create `.claude/cr/notes.md` with the header below if missing. Confirm on.
- `off` — delete `.claude/cr/learn.enabled`. **Keep `notes.md`.** Confirm off, notes preserved.
- `status` — report on/off, note count, last-updated date.
- `note <text>` — append a user-dictated note (`source: user`). Allowed even while off.
- `show` — print notes grouped by category.
- `forget <id>` — remove/retract that note.

## Notes schema

`.claude/cr/notes.md` starts with:

```
# Learned Notes
<!-- Written by review passes when learning is enabled. Terse, durable facts only. -->
```

Each note:

```
### N<id> · <category> · <ISO-date> · source: auto|user
<one or two lines of the lesson> — <file:path or area if code-specific>
```

Categories: `convention`, `false-positive`, `hotspot`, `arch`, `gotcha`.

## What counts as a good note

Only things that were **non-obvious**, will **recur**, and **aren't already in the code, git history, or codebase map**. Good: "raw SQL in `reports/export.ts` is safe — fixed allowlist, not user input; stop flagging it." Bad: restating what code does, one-off task state, anything trivially re-readable. When in doubt, don't write it.

## Behavior in review passes

When `.claude/cr/learn.enabled` exists, your review passes should (1) read `notes.md` at the start and fold it in — honor `false-positive` notes, scrutinize `hotspot` areas — and (2) append genuinely new durable lessons at the end, deduped. When off, neither read nor write it.

Perform the action, then report resulting state and the path to `notes.md`. Note that `.claude/cr/notes.md` is committed by default (add `.claude/cr/` to `.gitignore` to keep it local).
