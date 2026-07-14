# NemoHire

An AI-powered hiring copilot for Claude Code and Cowork. NemoHire builds a persistent professional identity for you, then uses it to source jobs and apply to them — in your own voice, submitting automatically once every question has a real answer — while keeping a local tracker of everything you've applied to.

## Why identity-first

Generic "write me a cover letter" tools produce generic cover letters. NemoHire's first step (`/nemohire:init`) builds a deep, persistent model of who you are — your real experience, achievements, salary rules, writing voice, and reusable interview answers — stored as plain markdown in `./.claude/nemohire/identity/`. Everything else reads from this before generating anything, so materials stay grounded in what's actually true about you, and sound like you.

## Commands

| Command | Purpose |
|---|---|
| `/nemohire:init` | Build/update your professional identity — runs directly in this session, no subagent needed for an interview |
| `/nemohire:source` | Find jobs from sites, keywords, roles, locations |
| `/nemohire:init-tracker` | Create the local application tracker |
| `/nemohire:apply` | Apply to jobs in bounded batches (default 5, `--batch-size`) — each job goes through `apply-agent` in a single dispatch: browse, write content, fill, submit, record locally |
| `/nemohire:continue` | Resume an interrupted `/nemohire:apply` run from its last checkpoint, one batch at a time |
| `/nemohire:sync-email` | Classify hiring emails and update the local tracker |
| `/nemohire:sync-tracker` | Mirror the local tracker into Notion, on demand — the only place Notion is ever touched |

Every agent runs on whatever model you've selected for the session — there's no hardcoded per-agent model split.

## Architecture — kept deliberately small

- **Commands** (`commands/*.md`) — user-facing entry points.
- **Agents** (`agents/*.md`) — 5 single-responsibility subagents:
  - **`apply-agent`** — the only agent that touches a job application. One dispatch per job: opens it, writes any cover letter/answers it needs in your voice, fills, uploads, submits, and updates the local tracker directly.
  - **`apply-batch-agent`** — runs one bounded batch of jobs in an isolated context, dispatching `apply-agent` once per job and checkpointing as it goes.
  - **`job-source-agent`** — finds and extracts postings for `/nemohire:source`.
  - **`tracker-sync-agent`** — mirrors the local tracker into Notion, only when `/nemohire:sync-tracker` is run.
  - **`email-agent`** — classifies hiring emails and updates the local tracker directly.
- **Skills** (`skills/*/SKILL.md`) — browser-navigation, chrome-connector, file-upload, notion-tracker, document-generation — reusable conventions agents read rather than re-implementing.
- **Memory** (`./.claude/nemohire/` in your project) — plain files, kept project-specific only (see `config.md` below). See `templates/` for schemas.

There's no separate browsing agent, content-writing agent, or mechanical file-write layer standing between these — each agent does its whole job itself, in one dispatch.

```
./.claude/nemohire/            # relative to the project root (the dir containing .claude/) — not ~/.claude
├── identity/                # who you are (14 files) — built by /nemohire:init
├── jobs/
│   ├── jobs.jsonl              # the jobs ledger — one line per job, filterable/editable per row
│   ├── details/<id>.md         # one file per job: posting, company context, cover letter, answers, record
│   └── resumes/<id>.<ext>      # tailored resumes only, if per-job tailoring is on
├── tracker/                 # applications.md (canonical, always) + sync-state.json (Notion link, if enabled)
├── emails/                  # last-sync watermark
├── browser-fallback-sites.json  # hostnames Playwright MCP has genuinely failed to load before
└── config.md                # project-specific only: Notion link/status, Gmail connector status, paths
```

### One ledger file, one details file per job

Every job — sourced or applied — lives as one line in `jobs/jobs.jsonl` (see `templates/tracker/jobs-ledger-schema.md`), plus one `jobs/details/<id>.md` holding its posting, company context, cover letter, answers, and application record. This replaced an earlier design with up to five separate files per job and a nested run-state JSON tree: updating one job used to mean reading and rewriting an entire structure, and finding a job's data meant knowing which of several folders to check. Now every path is deterministic from the job's `id`, every row is independently addressable by line, and filtering ("give me the pending jobs") is a `Grep` with a `head_limit`, never a full-file read.

### Sourcing and applying share the same ledger

`/nemohire:source` appends rows with `st:"new"` directly to `jobs.jsonl`, writing each posting's details file as it goes. `/nemohire:apply --jobs-file <path>` accepts that same ledger, or a hand-written file, or an export from somewhere else — for entries not already in the ledger, it mints fresh rows itself. You don't need to run `/nemohire:source` at all.

### Ids are minted directly by the top-level command, never searched for afterward

Before dispatching any batch, `/nemohire:apply` itself (running directly in this session, not a subagent) mints a fresh id for each selected job and writes its details file's Posting section — a mechanical file write, not something requiring the model to reason over the content. That's the one place arbitrary input format gets absorbed; everything downstream (`apply-agent` reading the posting, opening the application, writing content) works off that one file, by id, at one fixed path: `jobs/details/<id>.md`. Every agent that touches a job's data is told explicitly to treat that path as deterministic and to never fall back to scanning the filesystem if something looks missing — a real gap gets reported as a failure, not searched for. Doing the minting at the top level, as a plain file write, also keeps `apply-batch-agent`'s own dispatch cheap — it never has to ingest a job's full description just to pass it along.

Every job's details file carries a complete application record once submitted: the exact resume/cover-letter content, every question asked and answered, the company context, files uploaded, timestamp/URL. Since ids aren't stable across projects moved or re-sourced, duplicate-application checking is done by application/posting URL against the tracker directly, not by id.

### `/nemohire:apply` never re-applies to a job you've already applied to

Before showing you anything, `/nemohire:apply` reads `tracker/applications.md` directly (a plain file read, no agent dispatch needed) and drops any entry whose URL is already there, after normalizing scheme/case/trailing-slash/tracking params. The tracker is updated immediately after each job by `apply-agent` itself, not batched until the end — so this filter stays accurate even across an interrupted run.

### Local storage is unconditional; Notion is a fully separate, on-demand command

`/nemohire:init-tracker` creates `./.claude/nemohire/tracker/applications.md` — the canonical record every command reads and writes, always, regardless of Notion. Notion sync isn't something `/nemohire:apply` or `/nemohire:sync-email` ever waits on or triggers automatically — it happens only when you run `/nemohire:sync-tracker`, which asks (the first time) whether to link an existing database or create one, then mirrors every local row. Losing the connector, revoking access, or never connecting it doesn't break anything else in NemoHire.

### `/nemohire:apply` checkpoints every finished job, so a run is never lost

Each job, as soon as `apply-agent` returns an outcome — submitted, failed, needs_input, or manual — gets its one line in `./.claude/nemohire/jobs/jobs.jsonl` edited to that terminal `st` value immediately, before the next job starts, via a single-line `Edit` (never a rewrite of the whole file). If a run stops for any reason, nothing earlier is lost or redone — the ledger itself is the checkpoint, there's no separate run-state file to keep in sync with it.

### `/nemohire:continue` resumes exactly where a run stopped

It `Grep`s `jobs.jsonl` for rows still `"st":"queued"`, skips every job already terminal (submitted, failed, needs_input, or manual — nothing is silently retried), and runs the same batch sequence `/nemohire:apply` uses, starting from the first still-queued row — the same flow, entered from a different starting point.

### Email verification codes are retrieved via the Gmail connector, never guessed

Some forms email a one-time code or confirmation link after submission. `/nemohire:init` asks whether to connect the Gmail connector for this. If connected, `apply-agent` retrieves the code directly and confirms the page shows the application complete before treating it as submitted; if not, the job is flagged as needing that input instead.

## Browser strategy

`apply-agent`/`job-source-agent` use **Playwright MCP** by default, for every job — including logging in, which happens right in the same Playwright browser window a site is already open in, not by switching tools. The **Chrome Connector** is a reactive fallback only, reserved for two narrower cases: the page genuinely won't load in Playwright at all (recorded in `./.claude/nemohire/browser-fallback-sites.json` so future jobs on that domain skip straight to the Chrome Connector instead of failing Playwright again first), or a login genuinely can't be done through Playwright's own session (not interactive, or the site specifically needs the user's own already-authenticated browser profile). Never a hardcoded list of site names, never reached for just because a page is slower under one tool, never switched to silently, and never the automatic response to a login wall — that's handled in Playwright by default. See `skills/browser-navigation/SKILL.md` and `skills/chrome-connector/SKILL.md`.

There is exactly **one browser session, driven by one dispatch at a time** — `apply-batch-agent` never has two `apply-agent` calls (or two batches) touching the browser together.

Reading a page and filling its form uses **Playwright's structured page read** (its accessibility/element snapshot) rather than screenshots — most form-filling never needs vision at all. A screenshot is a fallback, only when the structured read genuinely can't disambiguate something. File uploads work differently per tool: in Playwright, the file is attached directly against the input element, no OS dialog involved; in the Chrome Connector (which doesn't have that capability), a native OS file picker is expected to open, and desktop control is used to select the file in it before handing control back to the browser — see `skills/file-upload/SKILL.md`.

Each job also has a **hard per-job action ceiling** (roughly 20 browser actions) so one stubborn multi-step form can't run away with an outsized share of a run's budget — past that, the job is flagged for a closer look instead.

## Safety guarantees for `/nemohire:apply`

- Within and across batches, only one browsing dispatch is ever in flight at a time — the underlying browser session is shared and single-threaded by design.
- No fields are ever filled with fabricated information; anything not grounded in identity or the posting is flagged `needs_input` for that job rather than guessed.
- A resume is always attached when the form has anywhere to put one — never skipped because a field wasn't marked required.
- A login wall is never treated as a reason to skip a job — you're asked to log in right there in Playwright and the same job continues from there; only genuine account **creation** or a multi-step process beyond a normal one-page form is flagged `manual` and left for you to finish by hand.
- There is **no pre-submit summary or confirmation gate** — `apply-agent` submits directly once everything required has a real answer. The full record is preserved in `jobs/details/<id>.md`'s Application record section for after-the-fact review.
- Failed submissions leave the tracker status untouched.

### The apply loop, per job — one dispatch, start to finish

0. Before the run starts, this session reads `tracker/applications.md` directly and drops jobs already applied to, then mints an id and writes `jobs/details/<id>.md`'s Posting section (plus a `st:"queued"` row in `jobs.jsonl`) for each selected job — a direct file write, not a subagent dispatch.
1. `apply-batch-agent` is dispatched per batch with just each job's `{id, application_url}` — it never sees a posting or company context in full.
2. For each job, one at a time: **dispatch `apply-agent`**. It opens the application, clicks through to the real form first if the page it lands on isn't already the form, decides whether it's a normal one-page form or one that needs manual handling (flagging it if so), writes a short company highlight and any cover letter/answers directly in your voice, fills, always attaches a resume when the form has anywhere to put one, handles an email verification step if needed, and submits.
3. `apply-agent` appends the rest of `jobs/details/<id>.md` (company context, cover letter, answers, application record) in a single write, and updates `tracker/applications.md` itself, directly.
4. `apply-batch-agent` checkpoints the outcome as a single-line edit to that job's row in `jobs.jsonl` right away, then moves to the next job.

One dispatch per job covers the whole thing — there's no hand-off between a browsing step and a separate content-writing step, and no separate file used purely to coordinate between two agents. `apply-batch-agent` itself stays a thin dispatcher: minting ids and seeding details files happens once, up front, at the top level — not repeated inside every batch dispatch.

### Human voice, always

Every piece of content that reaches an employer — resume (if tailored), cover letter, every live answer — is written strictly in your own first-person voice, in plain human grammar, with zero AI disclosure. See `skills/document-generation/SKILL.md`.

### A large queue doesn't need one marathon session

`/nemohire:continue` is meant to be re-invoked as many times as it takes — call it again yourself whenever convenient, or set it up as a recurring scheduled task (see the `schedule` skill). Every row's `st` value in `jobs.jsonl` makes stopping and restarting freely safe.

## Getting started

1. Run `/nemohire:init` to build your identity.
2. Run `/nemohire:init-tracker` to create the local tracker.
3. Run `/nemohire:source` with your target sites/keywords (optional — skip straight to step 4 with any jobs file you already have).
4. Run `/nemohire:apply` on the jobs you want to pursue.
5. If a run stops partway through, run `/nemohire:continue` to pick up exactly where it left off.
6. Run `/nemohire:sync-email` periodically to keep the tracker current.
7. Run `/nemohire:sync-tracker` any time you want the local tracker mirrored into Notion.

See `INSTALL.md` for setup in Claude Code vs. Cowork.

## Extending NemoHire

- Add a new job board or ATS quirk? Extend `skills/browser-navigation/SKILL.md` — commands stay stable.
- Want an additional sync target alongside local storage? Follow the `tracker-sync-agent`/`notion-tracker` pattern: local remains canonical, the new target is a separate on-demand command, never something `/nemohire:apply` waits on.
- New identity dimension? Add a template under `templates/identity/`, wire it into `commands/init.md`'s interview.
- Tempted to split `apply-agent` back into a browsing agent and a content-writing agent? Don't, unless you've hit a real, specific limit — the single-agent-per-job design is deliberate, both for simplicity and because it cuts dispatch count per job.
- Changing the per-job apply sequence? Edit it once in `commands/apply.md` — `commands/continue.md` points back at the same flow rather than duplicating it.

Contributions should keep each new command/agent/skill single-responsibility and keep the agent count small — that's what keeps this plugin maintainable.
