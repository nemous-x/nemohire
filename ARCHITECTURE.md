# NemoHire — Architecture

This is the technical reference for how NemoHire is built. If you just want to use the plugin, see `README.md` instead — this file is for anyone extending, debugging, or reviewing the plugin itself.

## Architecture — kept deliberately small

- **Commands** (`commands/*.md`) — user-facing entry points.
- **Agents** (`agents/*.md`) — 4 single-responsibility subagents, each scoped to only the tools it actually uses:
  - **`apply-agent`** — the only agent that touches a job application. One dispatch per **batch** of jobs (`--batch-size`, default 5, dynamic), dispatched directly by `/nemohire:apply`/`/nemohire:continue` (no intermediate batching agent): for each job in its batch, strictly one at a time, opens it, writes any cover letter/answers it needs in your voice, fills, uploads, submits, and writes that job's own details file — before moving to the next job. It never touches the tracker or any ledger row itself — the dispatching command writes both from `apply-agent`'s returned outcomes, once per job, right after the whole batch returns. Carries only Playwright MCP by default — the Chrome Connector and Gmail connector are deliberately excluded unless a specific job's `needs_input` outcome asks for one and you approve it.
  - **`job-source-agent`** — finds and extracts postings for `/nemohire:source`.
  - **`tracker-sync-agent`** — mirrors the local tracker into Notion, only when `/nemohire:sync-tracker` is run.
  - **`email-agent`** — classifies hiring emails and updates the local tracker directly.
- **Skills** (`skills/*/SKILL.md`) — browser-navigation, chrome-connector, file-upload, notion-tracker, document-generation — reusable conventions agents read rather than re-implementing.
- **Memory** (`./.claude/nemohire/` in your project) — plain files, kept project-specific only (see `config.md` below). See `templates/` for schemas.

There's no separate browsing agent, content-writing agent, or mechanical file-write layer standing between these — each agent does its whole job itself, in one dispatch.

```
./.claude/nemohire/            # relative to the project root (the dir containing .claude/) — not ~/.claude
├── identity/                # who you are (15 files) — built by /nemohire:init
├── jobs/
│   ├── jobs.jsonl              # the jobs ledger — one line per job, filterable/editable per row
│   ├── details/<seq>-<id>.md   # one file per job, created only once apply-agent finishes it
│   └── resumes/<seq>-<id>.<ext> # tailored resumes only, if per-job tailoring is on
├── tracker/                 # applications.md (canonical, always) + sync-state.json (Notion link, if enabled)
├── emails/                  # last-sync watermark
├── browser-fallback-sites.json  # hostnames Playwright MCP has genuinely failed to load before
└── config.md                # project-specific only: Notion link/status, Gmail connector status, paths
```

### One ledger file, one details file per job — the details file only appears once the job is actually done

Every job — sourced or applied — lives as one line in `jobs/jobs.jsonl` (see `templates/tracker/jobs-ledger-schema.md`). `jobs/details/<seq>-<id>.md`, holding its posting, company context, cover letter, answers, and application record, is created exactly once — by `apply-agent`, in a single write, right when it finishes that job. Nothing pre-seeds it at source or mint time: a job sitting as `new` or `queued` has no details file yet, on purpose. This replaced an earlier design with up to five separate files per job (some written before the job was even worked), a nested run-state JSON tree, and pre-seeded posting stubs that went stale if a listing changed or disappeared before the job was actually applied to. Now every path is deterministic from the job's `seq`+`id`, every row is independently addressable by line, `jobs/details/` only ever contains real, finished work, and filtering ("give me the pending jobs") is a `Grep` with a `head_limit`, never a full-file read.

### Sourcing and applying share the same ledger

`/nemohire:source` appends rows with `st:"new"` directly to `jobs.jsonl` — role, company, and both URLs, nothing more; no details file yet, since nothing's been applied to. `/nemohire:apply --jobs-file <path>` accepts that same ledger, or a hand-written file, or an export from somewhere else — for entries not already in the ledger, it mints fresh rows itself, the same way. You don't need to run `/nemohire:source` at all.

### Ids are minted directly by the top-level command, never searched for afterward

Before dispatching anything, `/nemohire:apply` itself (running directly in this session, not a subagent) mints a fresh id and `seq` for each selected job and appends its ledger row — a mechanical edit, not something requiring the model to reason over posting content. That's the one place arbitrary input format gets absorbed; everything downstream (`apply-agent` opening the application, reading the live posting, writing content) works off that one row, by id, and writes to one fixed, deterministic path once it's done: `jobs/details/<seq>-<id>.md`. Every agent that touches a job's data is told explicitly to treat that path as deterministic and to never fall back to scanning the filesystem if something looks missing — a real gap gets reported as a failure, not searched for. Doing the minting at the top level, as a plain ledger edit, also keeps each `apply-agent` dispatch cheap — it never has to ingest more than a batch's worth of `{id, seq, application_url}` triples just to get started.

### `apply-agent` is dispatched in batches, not one job at a time

Creating a subagent has a fixed cost, paid once per dispatch no matter how much work that dispatch does. Rather than pay that cost once per job, `/nemohire:apply`/`/nemohire:continue` hand `apply-agent` a batch of jobs at once (`--batch-size`, default 5, dynamic) as an array of `{id, seq, application_url}` triples, and `apply-agent` works through the whole batch itself, one job at a time, strictly sequentially — finishing one job's outcome and details file completely before opening the next job's `application_url`. This changes nothing about correctness (jobs were always applied to one at a time, sharing one browser session) — it just means a 20-job run is 4 dispatches at the default batch size instead of 20. If a batch is interrupted partway through (a browser session that can't recover, for instance), `apply-agent` reports real outcomes for whatever it finished and flags the untouched remainder of that batch `needs_input` so `/nemohire:continue` picks them back up in a fresh dispatch — nothing is silently lost.

Every job's details file carries a complete application record once submitted: the exact resume/cover-letter content, every question asked and answered, the company context, files uploaded, timestamp/URL — all gathered live by `apply-agent` itself, off the real page, not from a stale pre-extracted copy. Since ids aren't stable across projects moved or re-sourced, duplicate-application checking is done by application/posting URL against the tracker directly, not by id.

### `/nemohire:apply` never re-applies to a job you've already applied to

Before showing you anything, `/nemohire:apply` reads `tracker/applications.md` directly (a plain file read, no agent dispatch needed) and drops any entry whose URL is already there, after normalizing scheme/case/trailing-slash/tracking params. The tracker is updated immediately after each job — by `/nemohire:apply`/`/nemohire:continue` themselves, right after `apply-agent` returns, not batched until the end — so this filter stays accurate even across an interrupted run.

### Local storage is unconditional; Notion is a fully separate, on-demand command

`/nemohire:init-tracker` creates `./.claude/nemohire/tracker/applications.md` — the canonical record every command reads and writes, always, regardless of Notion. Notion sync isn't something `/nemohire:apply` or `/nemohire:sync-email` ever waits on or triggers automatically — it happens only when you run `/nemohire:sync-tracker`, which asks (the first time) whether to link an existing database or create one, then mirrors every local row. Losing the connector, revoking access, or never connecting it doesn't break anything else in NemoHire.

### `/nemohire:apply` checkpoints every finished job, so a run is never lost

Each job, as soon as `apply-agent` returns an outcome — submitted, failed, needs_input, or manual — gets its one line in `./.claude/nemohire/jobs/jobs.jsonl` edited to that terminal `st` value immediately, before the next job starts, via a single-line `Edit` (never a rewrite of the whole file). If a run stops for any reason, nothing earlier is lost or redone — the ledger itself is the checkpoint, there's no separate run-state file to keep in sync with it.

### `/nemohire:continue` resumes exactly where a run stopped

It `Grep`s `jobs.jsonl` for rows still `"st":"queued"`, skips every job already terminal (submitted, failed, needs_input, or manual — nothing is silently retried), and runs the same one-job-at-a-time sequence `/nemohire:apply` uses, starting from the first still-queued row — the same flow, entered from a different starting point.

### Email verification codes require explicit Gmail connector approval

Some forms email a one-time code or confirmation link after submission. `apply-agent` doesn't carry the Gmail connector by default (see "Browser strategy" below) — if a form needs one, it stops that job as `needs_input` naming the Gmail connector explicitly, rather than trying to work around not having it.

## Browser strategy

`apply-agent` uses **Playwright MCP only** by default — no Chrome Connector, no Gmail connector. This is deliberate: both are genuinely rare fallbacks, and granting either "just in case" would mean every dispatch pays for their tool schemas even on the vast majority of jobs that need neither. `job-source-agent` does carry the Chrome Connector (sourcing hits load failures often enough that it's worth having on hand), but not `apply-agent`.

Logging in happens right in the same Playwright browser window a site is already open in, not by switching tools — that's the default for every login, for every job. If `apply-agent` hits one of the two narrow cases the Chrome Connector exists for — the page genuinely won't load in Playwright at all (recorded in `./.claude/nemohire/browser-fallback-sites.json` so future jobs on that domain skip the doomed retry), or a login genuinely can't be done through Playwright's own session — it can't switch tools itself; it stops that job `needs_input`, naming the Chrome Connector explicitly, and the user decides whether to approve it (a deliberate, explicit edit to `apply-agent`'s own tool list — see `commands/apply.md`'s "Browser strategy" section for exactly how). Never a hardcoded list of site names, never granted preemptively, never the automatic response to a login wall. See `skills/browser-navigation/SKILL.md` and `skills/chrome-connector/SKILL.md`.

There is exactly **one browser session per `apply-agent` dispatch, driven one job at a time, strictly sequentially** — `apply-agent` never has two jobs from its own batch touching the browser together, and `/nemohire:apply`/`/nemohire:continue` never have two `apply-agent` dispatches (i.e. two batches) touching the browser at once either.

Reading a page and filling its form uses **Playwright's structured page read** (its accessibility/element snapshot) rather than screenshots — most form-filling never needs vision at all. A screenshot is a fallback, only when the structured read genuinely can't disambiguate something. File uploads go through Playwright's native file-attach action against the input element directly — no OS dialog ever appears — see `skills/file-upload/SKILL.md`.

Each job also has a **hard per-job action ceiling** (roughly 20 browser actions), reset for every job in a batch, so one stubborn multi-step form can't run away with an outsized share of a run's budget — past that, that job is flagged for a closer look instead and `apply-agent` moves on to the next job in its batch.

## Safety guarantees for `/nemohire:apply`

- Only one browsing dispatch is ever in flight at a time, and within that dispatch only one job is ever being worked on at a time — the underlying browser session is shared and single-threaded by design, across an entire batch, not just within one job.
- No fields are ever filled with fabricated information; anything not grounded in identity or the posting is flagged `needs_input` for that job rather than guessed.
- A resume is always attached when the form has anywhere to put one — never skipped because a field wasn't marked required.
- A login wall is never treated as a reason to skip a job — you're asked to log in right there in Playwright and the same job continues from there; only genuine account **creation** or a multi-step process beyond a normal one-page form is flagged `manual` and left for you to finish by hand.
- A missing Chrome Connector or Gmail connector is never worked around — the job stops `needs_input`, naming exactly which connector it needs, so enabling either is always something you explicitly approve.
- There is **no pre-submit summary or confirmation gate** — `apply-agent` submits directly once everything required has a real answer. The full record is written once, at the end of that job's own pass, into `jobs/details/<seq>-<id>.md`'s Application record section for after-the-fact review.
- Failed submissions leave the tracker status untouched.
- One job's outcome, content, or failure never affects how the next job in the same batch is worked — each job gets a clean, independent pass through the flow.

### The apply loop, per batch — one dispatch, several jobs, start to finish

0. Before the run starts, this session reads `tracker/applications.md` directly and drops jobs already applied to, then mints an id and `seq` and appends a `st:"queued"` row to `jobs.jsonl` for each selected job — a direct ledger edit, not a subagent dispatch, and no details file is touched.
1. Jobs are pulled off the queue in batches (`--batch-size`, default 5, dynamic), dispatched directly by this session — no intermediate batching agent: **dispatch `apply-agent` once per batch** with an array of `{id, seq, application_url}` — it never sees a posting or company context in full going in, for any job. For each job in the batch, in order, it opens the application directly (there's nothing pre-written to read first), clicks through to the real form first if the page it lands on isn't already the form, decides whether it's a normal one-page form or one that needs manual handling (flagging it if so), writes a short company highlight and any cover letter/answers directly in your voice, fills, always attaches a resume when the form has anywhere to put one, and submits — before moving to the next job in the batch.
2. For each job, right as it finishes, `apply-agent` writes that job's `jobs/details/<seq>-<id>.md` for the first time — the whole file, every section, in one write. Once every job in the batch has a real outcome, `apply-agent` returns one short outcome per job (status, note, date if submitted), in order. It never touches the tracker or any ledger row itself, for any job.
3. This session checkpoints the whole batch at once: for every job in the returned outcome array, a single-line edit to that job's row in `jobs.jsonl`, and an append/update to its row in `tracker/applications.md` from the `co`/`role`/`url` it already had plus what `apply-agent` returned for that job — all done before pulling the next batch.

One dispatch per batch covers several jobs' whole flow, applied one at a time inside that single dispatch — there's no hand-off between a browsing step and a separate content-writing step, and no separate agent or file used purely to coordinate between two layers. Minting ids happens once, up front, as a ledger-only edit — not repeated per job or per batch, and never involving `jobs/details/`.

### Human voice, always

Every piece of content that reaches an employer — resume (if tailored), cover letter, every live answer — is written strictly in your own first-person voice, in plain human grammar, with zero AI disclosure. See `skills/document-generation/SKILL.md`.

### A large queue doesn't need one marathon session

`/nemohire:continue` is meant to be re-invoked as many times as it takes — call it again yourself whenever convenient, or set it up as a recurring scheduled task (see the `schedule` skill). Every row's `st` value in `jobs.jsonl` makes stopping and restarting freely safe.

## Extending NemoHire

- Add a new job board or ATS quirk? Extend `skills/browser-navigation/SKILL.md` — commands stay stable.
- Want an additional sync target alongside local storage? Follow the `tracker-sync-agent`/`notion-tracker` pattern: local remains canonical, the new target is a separate on-demand command, never something `/nemohire:apply` waits on.
- New identity dimension? Add a template under `templates/identity/`, wire it into `commands/init.md`'s interview.
- Tempted to split `apply-agent` back into a browsing agent and a content-writing agent, or back into a per-job dispatch, or a wrapper batching agent around it? Don't, unless you've hit a real, specific limit — the single-agent, batch-dispatched design is deliberate, both for simplicity and because it minimizes dispatch count for a whole run.
- Changing the per-job apply sequence? Edit it once in `commands/apply.md` — `commands/continue.md` points back at the same flow rather than duplicating it.

Contributions should keep each new command/agent/skill single-responsibility and keep the agent count small — that's what keeps this plugin maintainable.
