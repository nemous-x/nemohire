---
description: Set up the local application tracker (Notion sync is a separate command — see /nemohire:sync-tracker)
allowed-tools: Read, Write, Glob
---

# /nemohire:init-tracker — Set Up Tracker

Creates the local tracker — that's the whole job here. Notion is handled entirely by `/nemohire:sync-tracker`, on its own, whenever the user wants it; this command doesn't ask about it at all.

## Behavior

1. Check `./.claude/nemohire/tracker/applications.md` — relative to the current project's working directory, **not** `~/.claude` — with `Read`/`Glob`; don't assume its state from earlier in the conversation.
2. If it doesn't exist, create it from `templates/tracker/applications.md`.
3. If it already exists, say so and leave it untouched — never overwrite existing rows.
4. Confirm the path and that it's ready for `/nemohire:apply` to read and write.

That's it. If the user wants applications mirrored to Notion, point them to `/nemohire:sync-tracker` — it handles linking or creating the database itself, the first time it's run.
