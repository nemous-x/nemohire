# Job Applications Tracker (Local — Canonical)

This is the always-on, offline-safe record of every application. It's created unconditionally by `/nemohire:init-tracker` and read/written by every NemoHire command before anything else, regardless of whether Notion sync is enabled — see `./.claude/nemohire/tracker/sync-state.json`. If Notion sync is on, this file's rows are mirrored there too, but this file is never optional or secondary.

| Job ID | Role | Company | Date Applied | Job Posting | Status | Notes |
|---|---|---|---|---|---|---|

<!-- Status starts as one of apply-agent's own outcomes — Submitted, Failed, Needs Input, Manual —
     written automatically the moment that job finishes. From there it's yours to update by hand as
     things progress: Interview, Offer, Accepted, Rejected, Ignored. Nothing in NemoHire overwrites
     a status you've moved past Submitted. -->
<!-- Date Applied must be ISO-8601 (YYYY-MM-DD) -->
<!-- Job ID matches the id used in jobs/jobs.jsonl and jobs/details/<seq>-<id>.md for this job. It's
     minted fresh per job (see templates/tracker/jobs-ledger-schema.md), so dedup for this table —
     and for any Notion database it's synced to — is by Job Posting URL, not Job ID. -->
<!-- A linked Notion database may not have an equivalent Job ID column; that's fine, this column
     is local-only traceability and never required on the Notion side. -->

