# Job Applications Tracker (Local — Canonical)

This is the always-on, offline-safe record of every application. It's created unconditionally by `/nemohire:init-tracker` and read/written by every NemoHire command before anything else, regardless of whether Notion sync is enabled — see `./.claude/nemohire/tracker/sync-state.json`. If Notion sync is on, this file's rows are mirrored there too, but this file is never optional or secondary.

| Job ID | Role | Company | Date Applied | Job Posting | Status | Notes |
|---|---|---|---|---|---|---|

<!-- Status values: To Apply, Applied, Interview, Offer, Accepted, Rejected, Ignored -->
<!-- Date Applied must be ISO-8601 (YYYY-MM-DD) -->
<!-- Job ID matches the jobs/applied/<id>/ folder name for this apply attempt. It's minted fresh
     per apply-attempt (see templates/tracker/job-cache-schema.md), so dedup for this table — and
     for any Notion database it's synced to — is by Job Posting URL, not Job ID. -->
<!-- A linked Notion database may not have an equivalent Job ID column; that's fine, this column
     is local-only traceability and never required on the Notion side. -->

