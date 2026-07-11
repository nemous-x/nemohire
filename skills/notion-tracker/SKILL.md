---
name: notion-tracker
description: >
  Creates and maintains the NemoHire Notion application-tracking database. Use during
  /nemo:init-tracker when the user chooses Notion, and whenever tracker-agent needs to insert
  or update an application row.
---

# Notion Tracker

## Database creation
Create a database titled **"📋 Job Applications"** with exactly these fields — do not add extras unless the user explicitly asks:

| Field | Type | Notes |
|---|---|---|
| Role | Title | |
| Company | Text | |
| Date Applied | Date | ISO-8601 |
| Job Posting | URL | |
| Status | Select | Options: To Apply, Applied, Interview, Offer, Accepted, Rejected, Ignored |
| Notes | Text | |

Store the resulting database ID in `.claude/nemohire/tracker/sync-state.json` as `{"backend": "notion", "database_id": "..."}`.

## Inserting/updating
- Before inserting, query the database for an existing row matching Company + Role. If found, update it instead of creating a duplicate.
- Status should generally only move forward (To Apply → Applied → Interview → Offer → Accepted), except Rejected/Ignored which can apply at any stage, and explicit reversals reported by a human or an email classification (e.g. a rescinded offer).
- Dates must be ISO-8601 (`YYYY-MM-DD`).

## Failure handling
If the Notion connector isn't available or isn't authorized, report that clearly and stop — don't fall back to local markdown without the user choosing that explicitly (that's a decision for `/nemo:init-tracker`, not a silent substitution).
