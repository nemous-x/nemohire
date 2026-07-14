---
description: Apply to sourced (or externally supplied) jobs, processed in bounded batches so token cost doesn't compound across a large queue. Checkpoints every finished job to jobs.jsonl; resumable via /nemohire:continue.
argument-hint: "[job selectors...] | --all | --jobs-file <path> | --batch-size <n>"
allowed-tools: Task, Read, Write, Edit, Glob, Grep, WebFetch, mcp__playwright__*, mcp__claude-in-chrome__*, mcp__gmail__*
---

# /nemohire:apply — Apply to Jobs

This command's own sequencing runs directly in this session: read pending jobs, mint ledger rows, dispatch one `apply-batch-agent` call per batch. It never touches a posting or a form itself — every job's actual work (browsing, writing content, filling, submitting) happens inside `apply-agent`, one dispatch per job, run by `apply-batch-agent` inside its batch. See `agents/apply-batch-agent.md`, `agents/apply-agent.md`, and `templates/tracker/jobs-ledger-schema.md` for the full schema.

**Why batches:** a large run driven directly in one session keeps every job's browser output in that session's context for the whole run, and cost compounds job after job. Dispatching each batch fresh means this session's context only grows by one short summary per batch.

## Path resolution — check before doing anything else

Every path below is relative to **the project root — the directory that contains `.claude/`**, which is always the `Read`/`Write`/`Edit`/`Glob`/`Grep` working directory for this command. Never assume `.claude/nemohire/` itself is the working directory; always use the full path starting with `./.claude/nemohire/`.

## Preconditions

1. **Identity must exist and be populated.** `Read`/`Glob` `./.claude/nemohire/identity/profile.md`, `experience.md`, and `documents.md` — must have real content, not placeholders. If not, stop and point to `/nemohire:init`.
2. **The local tracker must exist.** Check `./.claude/nemohire/tracker/applications.md`. If missing, stop and point to `/nemohire:init-tracker`.
3. **Check for an unfinished run.** `Grep` `./.claude/nemohire/jobs/jobs.jsonl` for `"st":"queued"`. If any exist, tell the user and suggest `/nemohire:continue` instead — only start selecting fresh jobs if they explicitly say to proceed anyway.

## The flow

0. **Find pending jobs.** If `--jobs-file <path>` was given, read that file (accept anything roughly job-shaped: title, company, description, posting URL, application URL, best-effort) and mint a fresh ledger row for any entry not already in `jobs.jsonl` (match by URL). Otherwise, `Grep` `./.claude/nemohire/jobs/jobs.jsonl` for `"st":"new"` — these are what `/nemohire:source` already found and minted.
1. **Skip already-applied jobs.** `Read` `./.claude/nemohire/tracker/applications.md` directly (a plain file, no dispatch needed) and drop any entry whose application/posting URL is already there. Normalize URLs first (lowercase scheme+host, strip trailing slash, drop tracking query params) so near-duplicates don't slip through.
2. **Let the user pick** which of the remaining entries to apply to (or use `--all` / explicit selectors).
3. **Mint and seed, right here, before dispatching anything.** For every selected job that doesn't already have a ledger row and details file, mint its `id`, append one line to `./.claude/nemohire/jobs/jobs.jsonl` (`st:"queued"`), and write `./.claude/nemohire/jobs/details/<id>.md`'s Posting section (`Write` — a plain, mechanical file copy of whatever fields the entry had, no agent reasoning needed). For a job that came from `/nemohire:source` and is already `st:"new"`, just flip that row to `st:"queued"` with a single-line `Edit` — its details file already exists. This keeps `apply-batch-agent`'s own dispatch context minimal — see "Keeping batch dispatch cheap" below.
4. **Process in batches, one at a time.** While any row is still `"st":"queued"`: `Grep` `jobs.jsonl` for `"st":"queued"` with `head_limit: <batch-size>` (default 5) to get the next batch, and dispatch `apply-batch-agent` with just their `{id, application_url}` pairs — nothing else, no posting text, no company data. Wait for it to fully return — never have two batch dispatches in flight together, since they share one browser session — report its summary briefly (e.g. "Batch 3/13 done — 4 submitted, 1 needs_input"), then dispatch the next batch.
5. **Close out.** Once no row is `"st":"queued"` anymore, you're done — there's no separate run/history file to move anything into; every row's terminal `st` value already is the record.

## Keeping batch dispatch cheap

`apply-batch-agent` should cost almost nothing to start. The expensive part of a job — reading the full posting, browsing, writing content — belongs entirely to `apply-agent`, dispatched once per job. `apply-batch-agent` itself never reads a posting or company context in full; it only ever sees the `{id, application_url}` pairs handed to it here and the one-line outcome each `apply-agent` dispatch returns. Minting ids and seeding details files happens here, at the top level, as a mechanical file write — not inside `apply-batch-agent`, which is what kept an earlier version of this flow from ever having to ingest a job's full description just to pass it along.

## Browser strategy

`apply-agent` uses **Playwright MCP** by default, for browsing and for logging in — a login wall is handled right there, in the same Playwright browser window that's already open, not by switching tools. The Chrome Connector is a reactive fallback reserved for two narrower cases: either the page genuinely won't load in Playwright at all — in which case the domain is remembered in `./.claude/nemohire/browser-fallback-sites.json` so later jobs on that site skip straight to the Chrome Connector — or a login genuinely can't be done through Playwright's own session (not interactive, or the site needs the user's own already-authenticated browser profile specifically). See `skills/browser-navigation/SKILL.md`.

## A login wall is never a reason to skip a job

If a site needs you to log in, `apply-agent` asks you to do it right there in Playwright and keeps going with the same job once you've confirmed — it never treats "needs login" as a reason to give up on a job. Only if you genuinely aren't available to log in during this run does that job stop short, and even then it's flagged `needs_input` (not `manual`), so it stays queued and `/nemohire:continue` picks it back up the moment you've logged in — nothing is silently skipped.

## Manual handling and missing information

`manual` is reserved for what genuinely can't be automated: the application needs a **brand-new account created** (a signup flow, not a login), or a multi-step process well beyond a normal one-page form. That's for you to finish by hand. If a real question can't be answered from your identity or the posting, that job is flagged `needs_input` with exactly what's missing, rather than guessed. Neither stalls the rest of the batch.

## Email verification

If a site emails a one-time code or confirmation link after submission, `apply-agent` retrieves it through the Gmail connector (if connected), enters it, and confirms the page shows the application complete before treating it as submitted. If the connector isn't connected or the code never arrives, the job is flagged `needs_input` instead.

## Tracker

Every submitted, failed, needs_input, or manual outcome is written straight to `tracker/applications.md` by `apply-agent` as it happens. Notion is never touched here — run `/nemohire:sync-tracker` whenever you want the local record mirrored there.

## Running a large queue without one marathon session

For dozens of jobs, don't try to push through all of it in one non-stop session — invoke `/nemohire:continue` again for the next batch whenever convenient, or set it up as a recurring scheduled task (see the `schedule` skill). Every row's `st` value in `jobs.jsonl` makes stopping and restarting freely safe.

## Human voice, always

Every cover letter and answer is written strictly in the user's own first-person voice, grounded in `identity/`, with zero AI disclosure. See `skills/document-generation/SKILL.md`.

## Model

Every agent in this flow runs on whatever model is selected for the current session — there's no per-agent override.
