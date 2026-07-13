---
description: Apply to sourced (or externally supplied) jobs, processed in bounded batches so token cost doesn't compound across a large queue. Checkpoints every finished job to run-state.json; resumable via /nemohire:continue.
argument-hint: "[job selectors...] | --all | --jobs-file <path> | --batch-size <n>"
allowed-tools: Task, Read, Write, Glob, WebFetch, mcp__playwright__*, mcp__claude-in-chrome__*, mcp__gmail__*
---

# /nemohire:apply — Apply to Jobs

This command's own sequencing runs directly in this session: read the jobs file, build the queue, dispatch one `apply-batch-agent` call per batch. It never touches a posting or a form itself — every job's actual work (browsing, writing content, filling, submitting, updating the local tracker) happens inside `apply-agent`, one dispatch per job, run by `apply-batch-agent` inside its batch. See `agents/apply-batch-agent.md` and `agents/apply-agent.md`.

**Why batches:** a large run driven directly in one session keeps every job's browser output in that session's context for the whole run, and cost compounds job after job. Dispatching each batch fresh means this session's context only grows by one short summary per batch.

## Preconditions — check before doing anything else

Actually call `Read`/`Glob` against `./.claude/nemohire/` — relative to the current project's working directory, **not** `~/.claude` — right now, never assume state from earlier in the conversation or a previous run.

1. **Identity must exist and be populated.** `./.claude/nemohire/identity/profile.md`, `experience.md`, and `documents.md` must have real content, not placeholders. If not, stop and point to `/nemohire:init`.
2. **The local tracker must exist.** Check `./.claude/nemohire/tracker/applications.md`. If missing, stop and point to `/nemohire:init-tracker`.
3. **Check for an unfinished run.** Read `./.claude/nemohire/jobs/run-state.json`. If `current_run` still has any job `"status": "queued"`, tell the user and suggest `/nemohire:continue` instead — only start a fresh run if they explicitly say to abandon the incomplete one.

## The flow

0. **Read the jobs file** yourself — `--jobs-file <path>` if given, otherwise `jobs/sourced.json`. Accept anything roughly job-shaped: title, company, description, posting URL, application URL, best-effort.
1. **Skip already-applied jobs.** Read `tracker/applications.md` directly (it's a plain file — no dispatch needed for this) and drop any entry whose application/posting URL is already there. Normalize URLs first (lowercase scheme+host, strip trailing slash, drop tracking query params) so near-duplicates don't slip through.
2. **Let the user pick** which of the remaining entries to apply to (or use `--all` / explicit selectors).
3. **Start the run.** Mint a fresh `run_id` and write `jobs/run-state.json`'s `current_run` with every selected job at `"status": "queued"`.
4. **Mint ids and seed the cache yourself, right here, before dispatching anything.** For every selected job, mint its id and write `jobs/cache/<id>/posting.md` directly with `Write` — a plain file copy of whatever fields the entry had, no agent reasoning needed for this, it's a mechanical step. Record each id back into `run-state.json`'s `current_run` entry for that job. This keeps `apply-batch-agent`'s own dispatch context minimal — see "Keeping batch dispatch cheap" below.
5. **Process in batches, one at a time.** While any job is still `"status": "queued"`: take the next `--batch-size` (default 5) entries and dispatch `apply-batch-agent` with just their `{id, application_url}` pairs — nothing else, no posting text, no company data. Wait for it to fully return — never have two batch dispatches in flight together, since they share one browser session — report its summary briefly (e.g. "Batch 3/13 done — 4 submitted, 1 manual"), then dispatch the next batch.
6. **Close out.** Once every job is `"status": "done"`, move `current_run` into `run-state.json`'s `history`.

## Keeping batch dispatch cheap

`apply-batch-agent` should cost almost nothing to start. The expensive part of a job — reading the full posting, browsing, writing content — belongs entirely to `apply-agent`, dispatched once per job. `apply-batch-agent` itself never reads a posting or company context in full; it only ever sees the `{id, application_url}` pairs handed to it here and the one-line outcome each `apply-agent` dispatch returns. If minting ids and seeding cache files happened inside `apply-batch-agent` instead (an earlier version of this flow), every batch's dispatch had to ingest each job's full description just to write it back out — that's the token cost this step now avoids by doing it here, directly, as a mechanical file write.

## Browser strategy

`apply-agent` uses **Playwright MCP** by default, for browsing and for logging in — a login wall is handled right there, in the same Playwright browser window that's already open, not by switching tools. The Chrome Connector is a reactive fallback reserved for two narrower cases: either the page genuinely won't load in Playwright at all — in which case the domain is remembered in `./.claude/nemohire/browser-fallback-sites.json` so later jobs on that site skip straight to the Chrome Connector — or a login genuinely can't be done through Playwright's own session (not interactive, or the site needs the user's own already-authenticated browser profile specifically). Never a fixed list of site names, and never the automatic response to every login. See `skills/browser-navigation/SKILL.md`.

## A login wall is never a reason to skip a job

If a site needs you to log in, `apply-agent` asks you to do it right there in Playwright and keeps going with the same job once you've confirmed — it never treats "needs login" as a reason to give up on a job. Only if you genuinely aren't available to log in during this run does that job stop short, and even then it's flagged `"outcome": "needs_input"` (not `"manual"`), so it stays queued and `/nemohire:continue` picks it back up the moment you've logged in — nothing is silently skipped.

## Manual handling and missing information

`"outcome": "manual"` is reserved for what genuinely can't be automated: the application needs a **brand-new account created** (a signup flow, not a login), or a multi-step process well beyond a normal one-page form. That's for you to finish by hand. If a real question can't be answered from your identity or the posting, that job is flagged `"outcome": "needs_input"` with exactly what's missing, rather than guessed. Neither stalls the rest of the batch.

## Email verification

If a site emails a one-time code or confirmation link after submission, `apply-agent` retrieves it through the Gmail connector (if connected), enters it, and confirms the page shows the application complete before treating it as submitted. If the connector isn't connected or the code never arrives, the job is flagged `needs_input` instead.

## Tracker

Every submitted, failed, needs_input, or manual outcome is written straight to `tracker/applications.md` by `apply-agent` as it happens. Notion is never touched here — run `/nemohire:sync-tracker` whenever you want the local record mirrored there.

## Running a large queue without one marathon session

For dozens of jobs, don't try to push through all of it in one non-stop session — invoke `/nemohire:continue` again for the next batch whenever convenient, or set it up as a recurring scheduled task (see the `schedule` skill). `run-state.json` makes stopping and restarting freely safe.

## Human voice, always

Every cover letter and answer is written strictly in the user's own first-person voice, grounded in `identity/`, with zero AI disclosure. See `skills/document-generation/SKILL.md`.

## Model

Every agent in this flow runs on whatever model is selected for the current session — there's no per-agent override.
