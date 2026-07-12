# NemoHire Config

This file is created by `/nemo:init` under `.claude/nemohire/config.md` in the user's project.

This file — together with `jobs/run-state.json` — is the central reference every NemoHire command reads before doing anything: `/nemo:init` writes it, `/nemo:init-tracker` updates the tracker section, `/nemo:apply` and `/nemo:continue` read the tracker backend and identity precondition from it and read/write `run-state.json`, `/nemo:sync-email` reads the tracker backend. No command should infer this information independently or duplicate it in its own private file.

## Preconditions every command checks here first
- `/nemo:apply` and `/nemo:continue`: identity must be populated (`.claude/nemohire/identity/` has real content, not placeholders) and a tracker backend must be set below. `/nemo:apply` additionally checks `jobs/run-state.json` for an unfinished run before starting a new one.
- `/nemo:sync-email`: tracker backend must be set below.

## Tracker backend
- **Backend:** <!-- notion | local, set by /nemo:init-tracker -->

## Browser strategy
- Built-in browser tooling is always attempted first.
- Chrome Connector fallback requires explicit, per-site user permission — never silent.

## Model routing defaults
- The active session running `/nemo:apply` handles sequencing, id minting, and cache seeding directly — there is no separate coordinator subagent for this.
- Haiku (`browser-agent`, `tracker-agent`, `memory-agent`, `job-source-agent`, `email-agent`), dispatched via `Task`: browsing, form navigation, mechanical field lookups, file management, uploads, tracker/email updates. Never writes or answers content.
- Sonnet (`identity-agent` only), dispatched via `Task`: all writing, personalization, and judgment — identity building, resume tailoring, cover letters, and every live application answer.

## Browser scope
- Only browser-scoped tools are ever used (the built-in browser tooling, or the Chrome Connector fallback) for job-application sites. Full computer-use/desktop-control tools are never used anywhere in this plugin.

## Notion is connector-only
- Whenever the tracker backend is `notion`, all reads/writes go through the Notion connector MCP tools via `tracker-agent` — never through the browser, and never through `browser-agent`. If the connector isn't authorized, `tracker-agent` stops and reports it rather than falling back to any other access path.

## Sourcing and applying are decoupled
- `jobs/sourced.json` is a plain array with no fixed schema — `/nemo:apply --jobs-file <path>` accepts it or any other reasonably job-shaped file identically. There's no schema contract between `/nemo:source`'s output and what `/nemo:apply` requires.

## Minimal-communication rule
- Job ids are minted by the active session running `/nemo:apply` itself, per apply-attempt, at the moment it's about to dispatch `browser-agent` for that job — never read from the input file, and never delegated to a coordinator subagent. The moment an id is minted, the active session seeds `jobs/cache/<id>/posting.md` from whatever fields it found. From then on, large text (postings, company/product context) is written once, to files keyed by that id, and read directly by whichever agent needs it — never re-sent between agents on every turn. See `templates/tracker/job-cache-schema.md`.
- `identity-agent` always re-reads both cache files fresh, for the id given in the current call — never carrying context from one job's id into another's.

## Resuming an interrupted run
- `/nemo:apply` checkpoints each finished job (submitted/failed/skipped/needs_input) to `jobs/run-state.json` immediately, not batched at the end (schema: `templates/tracker/run-state-schema.md`).
- `/nemo:continue` reads that same file, resumes from the first job still `queued`, and uses the exact same per-job sequence `/nemo:apply` does — it is not a separate implementation.
- `/nemo:apply` refuses to start a second overlapping run while an unfinished one exists in `run-state.json` — it points to `/nemo:continue` instead.

## Paths
- Identity: `.claude/nemohire/identity/`
- Sourced jobs (optional, no fixed schema): `.claude/nemohire/jobs/sourced.json`
- Per-job cache (schema: `templates/tracker/job-cache-schema.md`): `.claude/nemohire/jobs/cache/<id>/posting.md`, `.claude/nemohire/jobs/cache/<id>/company-context.md`
- Applied jobs: `.claude/nemohire/jobs/applied/<id>/` (resume.md, cover-letter.md, application-record.md)
- Run checkpoint ledger (schema: `templates/tracker/run-state-schema.md`): `.claude/nemohire/jobs/run-state.json`
- Tracker: `.claude/nemohire/tracker/`
- Emails: `.claude/nemohire/emails/`
