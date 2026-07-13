---
description: Mirror the local application tracker into Notion, on demand
allowed-tools: Task, Read, Glob
---

# /nemohire:sync-tracker — Sync Tracker to Notion

Delegate to `tracker-sync-agent` (see `agents/tracker-sync-agent.md`). This is the only place Notion sync happens in NemoHire — nothing during `/nemohire:apply`, `/nemohire:continue`, or `/nemohire:sync-email` touches Notion; they only ever write the local tracker. Run this command whenever you want that local record mirrored.

## Behavior

1. **Check the local tracker exists.** `Read`/`Glob` `./.claude/nemohire/tracker/applications.md` — relative to the current project's working directory, **not** `~/.claude` — if it's missing, point the user to `/nemohire:init-tracker` first.
2. **Dispatch `tracker-sync-agent`.** On first use, it asks whether to link an existing Notion database or create a new one, and records the choice in `sync-state.json`. On every use, it mirrors every local row into the linked database, updating existing pages by Job Posting URL rather than duplicating them.
3. **Report** what happened: rows created, rows updated, and whether the Notion connector was available.

## Notes
- The local tracker is unaffected either way — it's already the complete, correct record before this runs. This command only ever adds a mirror on top of it.
- Safe to run as often as you like; re-running it just re-syncs whatever's changed locally since the last time.
