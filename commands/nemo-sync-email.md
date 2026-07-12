---
description: Read hiring-related emails since the last sync, classify them, and update the tracker
allowed-tools: Task, Read, Write, Glob
---

# /nemo:sync-email — Sync Hiring Emails

Delegate to the `email-agent`.

## Behavior

1. Read `.claude/nemohire/emails/last-sync.json` for the timestamp of the last sync (default: 30 days ago if absent).
2. Fetch emails newer than that timestamp from the connected mail source.
3. Classify each into: rejection, interview invite, recruiter response, assessment/take-home, offer, or other/not-hiring-related. Discard "other" without reporting details.
4. For every classified hiring email, update the corresponding row in the tracker (backend read from `config.md`'s Tracker backend section, mirrored in `tracker/sync-state.json` — same source of truth `/nemo:apply` and `/nemo:continue` use) — advance status, add a short note. If the backend is `notion`, this goes through the Notion connector MCP via `tracker-agent`, never the browser.
5. Update `.claude/nemohire/emails/last-sync.json` with the new timestamp.
6. **Output only a summary**, never full email contents:
   - X interviews
   - X rejections
   - X recruiter responses
   - X assessments
   - X pending responses / actions needed (e.g. "reply to XYZ recruiter", "schedule interview with ABC Corp")

## Model routing
Haiku for reading, classifying, and syncing. No Sonnet needed unless a message is genuinely ambiguous, in which case escalate just that one classification.
