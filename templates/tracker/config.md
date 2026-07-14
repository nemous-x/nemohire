# NemoHire Project Config

This file is created by `/nemohire:init` under `./.claude/nemohire/config.md` in this project. It holds only the values specific to *this* project — Notion sync status, the Gmail connector's connection state, and the paths NemoHire uses here. How the plugin behaves (agent roles, browser strategy, batching, model policy) is documented once in the plugin itself — `README.md`, `agents/`, `skills/` — not duplicated into every project's config.

## Tracker
- **Local storage:** always enabled — `./.claude/nemohire/tracker/applications.md` is the canonical record.
- **Notion sync:** <!-- off by default; set to "on" once /nemohire:sync-tracker links a database -->
- **Notion database:** <!-- URL, set by /nemohire:sync-tracker once linked -->

## Gmail connector
- **Status:** <!-- connected / not connected, set during /nemohire:init -->

## Browser fallback sites
- Sites Playwright MCP has failed to load before are recorded in `./.claude/nemohire/browser-fallback-sites.json` and consulted automatically — see `skills/browser-navigation/SKILL.md`. Nothing to configure here.

## Paths
- Identity: `./.claude/nemohire/identity/`
- Jobs ledger (one line per job — see `templates/tracker/jobs-ledger-schema.md`): `./.claude/nemohire/jobs/jobs.jsonl`
- Per-job details (one file per job): `./.claude/nemohire/jobs/details/<id>.md`
- Tailored resumes (only if per-job tailoring is on): `./.claude/nemohire/jobs/resumes/<id>.<ext>`
- Browser fallback memory: `./.claude/nemohire/browser-fallback-sites.json`
- Tracker: `./.claude/nemohire/tracker/`
- Emails: `./.claude/nemohire/emails/`
