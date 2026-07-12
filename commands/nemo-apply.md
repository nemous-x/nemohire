---
description: Apply to sourced (or externally supplied) jobs one at a time — prepares materials, answers every question in one batch pass via a shared file, then submits directly
argument-hint: "[job selectors...] | --all | --jobs-file <path>"
allowed-tools: Task, Read, Write, Glob
---

# /nemo:apply — Apply to Jobs

**This command's coordination logic runs directly in the active session — it is not delegated to a subagent.** There is no `application-coordinator-agent`. You (the active session) mint ids, seed cache files, sequence the steps below, and dispatch `identity-agent`, `browser-agent`, `tracker-agent`, and `memory-agent` via `Task` one at a time. You never draft an answer, a resume line, or a cover-letter sentence yourself — that stays exclusively `identity-agent`'s job — but the sequencing itself is yours to run, in-session, not something you hand off to another coordinator agent.

**Applications are processed strictly sequentially — never in parallel.** There's no separate ranking or prep step. Requires a tracker (`/nemo:init-tracker`) — the very first thing this command does is ask the tracker which jobs are already applied to, so it needs to know which backend is active (read from `config.md`).

## Any jobs file works — there's no schema link to `/nemo:source`

`--jobs-file <path>` accepts anything that roughly looks like a list of jobs: `.claude/nemohire/jobs/sourced.json` (from `/nemo:source`), a file you wrote by hand, an export from somewhere else. If `--jobs-file` isn't given, it defaults to `jobs/sourced.json`. Read it yourself, right here in the active session — don't require an id, a status, or exact field names, just pull out best-effort whatever's there per entry: title, company, description/requirements, posting URL, application URL.

## Notion access is connector-only, never browser

Whenever the active tracker backend is `notion`, every read and write to it — including the already-applied URL check below and the post-submit update — happens through `tracker-agent` using the Notion connector MCP tools. Neither you nor `browser-agent` ever opens notion.so in a browser tab or touches Notion through any browser-scoped or computer-use tool. If the connector isn't authorized, stop and tell the user, rather than trying any other path to Notion.

## Ids are minted per apply-attempt, and questions travel through a file, not through Task payloads

The moment you're about to actually apply to a specific job, you mint a fresh id yourself and write `jobs/cache/<id>/posting.md` from whatever you found for that entry. From there:

- **`browser-agent`** (Haiku, browser-scoped tools only — never full computer control): given `{id, application_url}`, opens the application, writes the company/product description straight to `jobs/cache/<id>/company-context.md`, fills anything it can already answer itself from `identity/profile.md`/`identity/documents.md`, and writes **every remaining question** to `jobs/cache/<id>/questions.md` in one pass — reporting back only a count, never the questions themselves.
- **`identity-agent`** (Sonnet): given just the `id`, reads `posting.md`, `company-context.md`, and `questions.md`, and writes an answer into `questions.md` for **every** question in one dispatch — not one dispatch per question. Always strictly in the user's own first-person voice, as the person, in plain human grammar, with zero AI disclosure. If it can't answer something, it writes `[NEEDS INPUT: ...]` instead of guessing.
- **`browser-agent`** again: reads the now-answered `questions.md`, fills every field, uploads the prepared files, and **submits directly** — no summary shown, no user confirmation waited on. If anything in `questions.md` is still flagged `[NEEDS INPUT: ...]`, it stops and reports exactly what's missing instead of guessing or submitting incomplete.
- **You** (the active session): mint the id, seed the posting cache, and dispatch the above in sequence via `Task`. You never hold question text, answers, or company context yourself for more than a moment — every one of those lives in a file, referenced only by id.

## The flow, concretely

1. **Read the jobs file**, yourself, in the active session.
2. **Skip already-applied jobs.** Dispatch `tracker-agent` (Task) to return every application/posting URL already in the tracker; remove any matching entry before doing anything else.
3. **Let the user pick** which of the remaining entries to apply to (or use `--all` / explicit selectors).
4. **Mint + seed.** For the job about to be applied to: mint a fresh `id` yourself, write `jobs/cache/<id>/posting.md` yourself.
5. **Prepare materials.** Dispatch `identity-agent` (Task) with just the `id` and the instruction "prepare materials for this posting." It reads the cached posting, always writes a cover letter, and only tailors a resume if `identity/documents.md` says the user opted into per-job tailoring during `/nemo:init` — otherwise it copies the base resume in unchanged. Both saved to `jobs/applied/<id>/`.
6. **Extract questions.** Dispatch `browser-agent` (Task) with `{id, application_url}`. It opens the application, caches company context, fills what it can itself, writes every remaining question to `jobs/cache/<id>/questions.md`, and reports just a count.
7. **Answer questions.** If there were any, dispatch `identity-agent` (Task) with just the `id` and the instruction to answer everything in `questions.md`, writing directly into that file.
8. **Handle missing info.** If any answer is flagged `[NEEDS INPUT: ...]`, stop and ask the user for just that — don't let anything downstream guess.
9. **Fill, upload, submit.** Dispatch `browser-agent` (Task) again to fill every field from `questions.md`, upload the documents, and submit — directly, without a pre-submit summary or confirmation step.
10. **Application record.** Dispatch `memory-agent` (Task) to write `jobs/applied/<id>/application-record.md`, compiled from the cache files and the materials.
11. **Update the tracker, right away.** Dispatch `tracker-agent` (Task) immediately after this job submits — don't batch tracker updates until the run ends — so the tracker (and the next run's already-applied filter) stays accurate even if the run stops partway through. Dispatch `browser-agent` to close the tab(s).

This repeats **per job, sequentially**, driven by you in the active session — not by a separate coordinator subagent.

## Browser strategy

Try the built-in browser tooling first. If a site can't be accessed that way, **stop and ask the user for explicit permission** before switching to the Chrome Connector (`skills/chrome-connector/SKILL.md`) — decided once per site, never silently. **No agent in this flow ever uses full computer/desktop-control tools — only browser-scoped tools.** This applies to job-application sites; Notion is handled exclusively through its connector MCP, never through the browser at all (see above).

## Safety and failure handling

- `browser-agent` never fills or submits a field it — or `identity-agent` — didn't have a real answer for; a `[NEEDS INPUT: ...]` flag always stops the run for that job rather than guessing.
- If submission fails (validation error, site error, CAPTCHA loop), stop, report the exact failure, and leave the tracker untouched for that job.
- There is no pre-submit summary or confirmation gate — submission happens as soon as every question has a real answer. Everything about the application (posting, company context, every question and answer, files uploaded) is preserved in `jobs/cache/<id>/` and `jobs/applied/<id>/application-record.md` for after-the-fact review.

## Human voice, always

Everything that reaches the live form — resume (if tailored), cover letter, every answer — is written strictly in the user's own first-person voice, as the candidate, in plain human grammar. Nothing may disclose, hint at, or reference AI involvement. Only `identity-agent` produces this content. See `skills/document-generation/SKILL.md`.

## Model routing

You (the active session running this command) handle sequencing, id minting, cache seeding, and reading the jobs file directly — zero content generation, and no separate coordinator subagent. Haiku (`browser-agent`, `tracker-agent`, `memory-agent`), dispatched via `Task`: navigation, extraction, filling, uploads, submit, tracker I/O, record-keeping — zero content generation. Sonnet (`identity-agent`), dispatched via `Task`: every piece of written content, produced in batches of one dispatch per job stage, not per question.
