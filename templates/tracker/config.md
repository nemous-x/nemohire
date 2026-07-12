# NemoHire Config

This file is created by `/nemo:init` under `.claude/nemohire/config.md` in the user's project.

## Tracker backend
- **Backend:** <!-- notion | local, set by /nemo:init-tracker -->

## Browser strategy
- Built-in browser tooling is always attempted first.
- Chrome Connector fallback requires explicit, per-site user permission — never silent.

## Model routing defaults
- Haiku: browsing, form navigation, mechanical field lookups, file management, uploads, tracker/email updates, coordination/relaying. Never writes or answers content.
- Sonnet (`identity-agent` only): all writing, personalization, and judgment — identity building, resume tailoring, cover letters, and every live application answer.

## Browser scope
- Only browser-scoped tools are ever used (the built-in browser tooling, or the Chrome Connector fallback). Full computer-use/desktop-control tools are never used anywhere in this plugin.

## Minimal-communication rule
- Every job has a stable, unique id the moment it enters `jobs.json`. Large text (job postings, company/product context) is written once, to a file keyed by that id, and read directly by whichever agent needs it — never re-sent between agents on every turn. See `templates/tracker/jobs-schema.md`.

## Paths
- Identity: `.claude/nemohire/identity/`
- Jobs list: `.claude/nemohire/jobs/jobs.json` (schema: `templates/tracker/jobs-schema.md`) — sourced and manually-imported jobs alike, tracked by `status`
- Per-job cache: `.claude/nemohire/jobs/cache/<id>/company-context.md`
- Applied jobs: `.claude/nemohire/jobs/applied/<id>/` (resume.md, cover-letter.md, application-record.md)
- Tracker: `.claude/nemohire/tracker/`
- Emails: `.claude/nemohire/emails/`
