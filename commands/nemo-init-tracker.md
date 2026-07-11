---
description: Set up the application tracker (Notion database or local markdown)
argument-hint: "[notion|local]"
allowed-tools: Task, Read, Write, Glob
---

# /nemo:init-tracker — Set Up Tracker

Delegate to the `tracker-agent`.

## Behavior

1. **Ask** (unless given as an argument): "Where should applications be tracked — Notion or local markdown?"
2. **If Notion:** confirm a Notion connector is available. If not, tell the user to connect Notion first and stop. Otherwise create a database titled "📋 Job Applications" with exactly these fields:
   - Role (title)
   - Company (text)
   - Date Applied (date, ISO-8601)
   - Job Posting (url)
   - Status (select: To Apply, Applied, Interview, Offer, Accepted, Rejected, Ignored)
   - Notes (text)

   Record the database ID in `.claude/nemohire/tracker/sync-state.json`.
3. **If local:** create `.claude/nemohire/tracker/applications.md` from `templates/tracker/applications.md` with the same columns as a markdown table, and initialize `.claude/nemohire/tracker/sync-state.json` with `{"backend": "local"}`.
4. **Never re-create** an existing tracker without asking first — check `sync-state.json`.
5. **Confirm** what was created and where.

## Model routing
Haiku — schema setup and file/database creation only.
