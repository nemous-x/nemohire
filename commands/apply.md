---
description: Apply to sourced (or externally supplied) jobs, dispatched to apply-agent in batches (default 5, dynamic) so subagent overhead is amortized across several jobs instead of paid per job. Checkpoints every finished job to jobs.jsonl; resumable via /nemohire:continue.
argument-hint: "[job selectors...] | --all | --jobs-file <path> | --batch-size <n>"
allowed-tools: Task, Read, Write, Edit, Glob, Grep, WebFetch, mcp__playwright__*, mcp__claude-in-chrome__*, mcp__gmail__*
---

# /nemohire:apply — Apply to Jobs

This command's own sequencing runs directly in this session: read pending jobs, mint ledger rows, dispatch `apply-agent` directly with a batch of jobs at a time (`--batch-size`, default 5), and checkpoint every job's outcome yourself right after each batch returns. There's no intermediate batching agent — `apply-agent` itself now takes the whole batch and works through it one job at a time, strictly sequentially, returning one short outcome per job. Dispatching a batch instead of a single job means the fixed cost of creating that subagent is paid once per batch, not once per job, while jobs are still applied to exactly as sequentially as before (they always shared one browser session anyway). See `agents/apply-agent.md` and `templates/tracker/jobs-ledger-schema.md` for the full schema.

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
3. **Mint, right here, before dispatching anything.** For every selected job that doesn't already have a ledger row, mint its `id`, compute its `seq` (current line count + 1), and append one line to `./.claude/nemohire/jobs/jobs.jsonl` (`st:"queued"`, `ref` precomputed as `./.claude/nemohire/jobs/details/<seq>-<id>.md`). For a job that came from `/nemohire:source` and is already `st:"new"`, just flip that row to `st:"queued"` with a single-line `Edit` — its `seq`/`ref` are already set. This is purely a ledger edit — `jobs/details/` is never touched here; that file doesn't exist until `apply-agent` finishes the job.
4. **Process the queue in batches, one `apply-agent` dispatch per batch.** While any row is still `"st":"queued"`: `Grep` `jobs.jsonl` for `"st":"queued"` with `head_limit: <batch-size>` (default 5, dynamic via `--batch-size`) to pull the next batch, noting each row's `co`, `role`, and `url` from that same `Grep` result — you'll need them in a moment. **Dispatch `apply-agent` once** with the whole batch as an array of `{id, seq, application_url}` objects — nothing else per job, no posting text, no company data. `apply-agent` works through the batch itself, one job at a time, strictly sequentially (see `agents/apply-agent.md`); you dispatch it once and wait for the whole batch to fully return — never a second dispatch touching the browser while one is still running. It returns one outcome per job, in the same order, each tagged with that job's `id`. The instant the batch returns, for **every** job in it, in order, do both of these before doing anything else:
   - `Grep` `jobs.jsonl` for `"id":"<id>"` to get that job's exact current line and `Edit` it to the terminal `st` (and `note` if relevant) — a single-line edit, one per job.
   - Append or update the matching row in `./.claude/nemohire/tracker/applications.md` yourself — Job ID (`id`), Role, Company, Date Applied (today, ISO-8601, only if the outcome is `submitted`), Job Posting URL, Status (the outcome), Notes (the note `apply-agent` returned). You already have Role/Company/URL from the ledger row and status/note from `apply-agent`'s return — no extra read needed. Dedupe by URL, same normalization as step 1.

   This leaves the run resumable exactly where it stopped even if interrupted mid-batch — a job `apply-agent` never got to simply stays `"st":"queued"` since nothing gets checkpointed until its batch actually returns. After a batch is checkpointed, report a short summary (e.g. "5/5 done — 4 submitted, 1 needs_input") before pulling the next batch.
5. **Close out.** Once no row is `"st":"queued"` anymore, you're done — there's no separate run/history file to move anything into; every row's terminal `st` value already is the record.

## Why batching happens inside apply-agent, not a wrapper agent

Creating any subagent carries a fixed overhead, paid once per dispatch regardless of how much work that dispatch actually does. An earlier version of this flow dispatched `apply-agent` once per job — for a queue of 20 jobs, that's 20 dispatches, each paying that fixed cost on top of the job's own work. A version before that used a thin `apply-batch-agent` that dispatched `apply-agent` per job inside it, which is strictly worse — two dispatches' worth of fixed overhead for every one job.

Now `apply-agent` itself takes a batch (`--batch-size`, default 5) and works through it one job at a time, strictly sequentially — the jobs were always going to share one browser session and run one after another anyway, so nothing about correctness or safety changes. What changes is that the fixed per-dispatch cost is now paid once per batch instead of once per job: a 20-job queue at the default batch size is 4 dispatches, not 20. Bigger batches amortize that cost further but make each dispatch take longer to return and risk more re-work if a batch is interrupted partway through (see step 4); smaller batches check in more often but pay overhead more times. 5 is the default balance — override with `--batch-size` if a particular run calls for something else.

## Browser strategy

`apply-agent` uses **Playwright MCP only** — it doesn't carry the Chrome Connector or Gmail connector by default, deliberately, so most jobs (which need neither) don't pay for those tools' schemas on every dispatch. Logging in happens right there in the same Playwright browser window that's already open, not by switching tools — that's the default for every login. If a job genuinely needs the Chrome Connector (the page won't load in Playwright at all, or a login genuinely can't be done through Playwright's own session) or the Gmail connector (an email verification code), `apply-agent` can't reach for either itself — it stops that job as `needs_input` with a note naming exactly which connector is needed. See `skills/browser-navigation/SKILL.md`.

**Resolving a needs_input job that names a missing connector** is a deliberate, explicit step, never automatic: tell the user which job needs which connector, and if they approve enabling it, that means adding it to `apply-agent`'s own `tools:` list in `agents/apply-agent.md` (a real edit, not a runtime toggle) before re-running `/nemohire:continue` on that job — or, for a one-off, handling that job's remaining step directly in this session, since this command's own toolset already includes both connectors. Either way, nothing reaches for Chrome or Gmail without the user having explicitly said yes to that specific job.

## A login wall is never a reason to skip a job

If a site needs you to log in, `apply-agent` asks you to do it right there in Playwright and keeps going with the same job once you've confirmed — it never treats "needs login" as a reason to give up on a job. Only if you genuinely aren't available to log in during this run does that job stop short, and even then it's flagged `needs_input` (not `manual`), so it stays queued and `/nemohire:continue` picks it back up the moment you've logged in — nothing is silently skipped.

## Manual handling and missing information

`manual` is reserved for what genuinely can't be automated: the application needs a **brand-new account created** (a signup flow, not a login), or a multi-step process well beyond a normal one-page form. That's for you to finish by hand. If a real question can't be answered from your identity or the posting, that job is flagged `needs_input` with exactly what's missing, rather than guessed. None of these stall the rest of the run.

## Tracker

Every submitted, failed, needs_input, or manual outcome is written straight to `tracker/applications.md` by this session, right after each batch's `apply-agent` dispatch returns, once per job in that batch — not by `apply-agent` itself (see step 4). Notion is never touched here — run `/nemohire:sync-tracker` whenever you want the local record mirrored there.

## Running a large queue without one marathon session

For dozens of jobs, don't try to push through all of it in one non-stop session — invoke `/nemohire:continue` again for the next chunk whenever convenient, or set it up as a recurring scheduled task (see the `schedule` skill). Every row's `st` value in `jobs.jsonl` makes stopping and restarting freely safe.

## Human voice, always

Every cover letter and answer is written strictly in the user's own first-person voice, grounded in `identity/`, with zero AI disclosure. See `skills/document-generation/SKILL.md`.

## Model

Every agent in this flow runs on whatever model is selected for the current session — there's no per-agent override.
