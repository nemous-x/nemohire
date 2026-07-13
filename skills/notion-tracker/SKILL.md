---
name: notion-tracker
description: >
  Notion sync layer for the NemoHire application tracker, used only by tracker-sync-agent when
  the user runs /nemohire:sync-tracker. Local markdown is always the canonical record, written
  directly by apply-agent/email-agent as applications happen — this skill never runs
  automatically alongside them.
---

# Notion Tracker (Sync Layer)

## On demand, not automatic

Notion sync only happens when the user runs `/nemohire:sync-tracker`. Nothing about `/nemohire:apply`, `/nemohire:continue`, or `/nemohire:sync-email` waits for or depends on this — the local tracker is already the complete, correct record before this skill ever runs. If the connector is temporarily unavailable when sync is requested, that run is simply reported as failed — the local tracker is unaffected either way.

## Connector-only, never browser (non-negotiable)

Every interaction with Notion — fetching a database's schema, inserting rows, updating status, querying existing rows/URLs — goes through the Notion connector MCP tools (`mcp__notion__*`, or the Claude Code equivalent). Never open notion.so through `apply-agent`, `job-source-agent`, or any browser-scoped tool, and never fall back to computer-use/desktop control. If the Notion connector isn't connected or isn't authorized, report that clearly and stop.

## Linking an existing database vs. creating a new one

The first time `/nemohire:sync-tracker` runs, `tracker-sync-agent` asks the user whether they already have a Notion database they want to sync to, or want a new one created.

- **Linking an existing database:** fetch it first via the connector to read its actual schema and data source id. **Do not assume the canonical field names or that every field exists.** Map to whatever the real database has: a title property (whatever it's called) for Role, and text/date/url/select properties for Company, Date Applied, Job Posting, Status, Notes wherever they exist by name or clear equivalent. If the database has no analogue for one of these fields, skip that field for Notion writes rather than trying to add columns to a database the user already curates and uses for other things — don't restructure someone else's existing database without being asked to. Record the database id and data source id in `sync-state.json`.
- **Creating a new database:** use the canonical schema — do not add extras unless the user explicitly asks:

  | Field | Type | Notes |
  |---|---|---|
  | Role | Title | |
  | Company | Text | |
  | Date Applied | Date | ISO-8601 |
  | Job Posting | URL | |
  | Status | Select | Options: To Apply, Applied, Interview, Offer, Accepted, Rejected, Ignored |
  | Notes | Text | |

  Store the resulting database id and data source id in `sync-state.json`.

## Inserting/updating

- Before inserting, query the database for an existing row matching the Job Posting URL (the same dedup key used locally). If found, update it instead of creating a duplicate.
- Status should generally only move forward (To Apply → Applied → Interview → Offer → Accepted), except Rejected/Ignored which can apply at any stage, and explicit reversals reported by a human or an email classification (e.g. a rescinded offer).
- Dates must be ISO-8601 (`YYYY-MM-DD`).

## Failure handling

If the Notion connector isn't available or isn't authorized at write time, report that clearly and move on — the local write already happened (or happens regardless), so nothing is lost. Never silently disable sync going forward because of one failed attempt; that's a decision for the user, not an automatic fallback.
