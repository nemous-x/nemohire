---
description: Read hiring-related emails since the last sync, classify them, and update the local tracker
allowed-tools: Task, Read, Write, Glob, mcp__gmail__*
---

# /nemohire:sync-email — Sync Hiring Emails

Delegate to `email-agent`. It updates `tracker/applications.md` directly — Notion is never touched here; run `/nemohire:sync-tracker` separately if you want the update mirrored there.

## Behavior

1. Read `./.claude/nemohire/emails/last-sync.json` — relative to the current project's working directory, **not** `~/.claude` — for the timestamp of the last sync (default: 30 days ago if absent).
2. Dispatch `email-agent` to fetch, classify, and update the local tracker directly for every hiring-related message.
3. **Output only the summary it returns**, never full email contents: interviews, rejections, recruiter responses, assessments, and any pending actions needed.

## Model

Runs on whatever model is selected for the current session.
