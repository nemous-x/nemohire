---
description: Apply to sourced (or externally supplied) jobs one at a time — prepares materials, answers every question in one batch pass via a shared file, then submits directly
argument-hint: "[job selectors...] | --all | --jobs-file <path>"
allowed-tools: Task, Read, Write, Glob
---

# /nemo:apply — Apply to Jobs

Orchestrated end-to-end by `application-coordinator-agent` (Haiku) — a pure relay that never writes, answers, or holds large text itself. **Applications are processed strictly sequentially — never in parallel.** There's no separate ranking or prep step. Requires a tracker (`/nemo:init-tracker`) — the very first thing this command does is ask the tracker which jobs are already applied to, so it needs to know which backend is active (read from `config.md`).

## Any jobs file works — there's no schema link to `/nemo:source`

`--jobs-file <path>` accepts anything that roughly looks like a list of jobs: `.claude/nemohire/jobs/sourced.json` (from `/nemo:source`), a file you wrote by hand, an export from somewhere else. If `--jobs-file` isn't given, it defaults to `jobs/sourced.json`.

## Ids are minted per apply-attempt, and questions travel through a file, not through Task payloads

The moment `application-coordinator-agent` is about to actually apply to a specific job, it mints a fresh id and writes `jobs/cache/<id>/posting.md` from whatever it found for that entry. From there:

- **`browser-agent`** (Haiku, browser-scoped tools only — never full computer control): given `{id, application_url}`, opens the application, writes the company/product description straight to `jobs/cache/<id>/company-context.md`, fills anything it can already answer itself from `identity/profile.md`/`identity/documents.md`, and writes **every remaining question** to `jobs/cache/<id>/questions.md` in one pass — reporting back only a count, never the questions themselves.
- **`identity-agent`** (Sonnet): given just the `id`, reads `posting.md`, `company-context.md`, and `questions.md`, and writes an answer into `questions.md` for **every** question in one dispatch — not one dispatch per question. Always strictly in the user's own first-person voice, as the person, in plain human grammar, with zero AI disclosure. If it can't answer something, it writes `[NEEDS INPUT: ...]` instead of guessing.
- **`browser-agent`** again: reads the now-answered `questions.md`, fills every field, uploads the prepared files, and **submits directly** — no summary shown, no user confirmation waited on. If anything in `questions.md` is still flagged `[NEEDS INPUT: ...]`, it stops and reports exactly what's missing instead of guessing or submitting incomplete.
- **`application-coordinator-agent`** (Haiku): mints the id, seeds the posting cache, and dispatches the above in sequence. It never holds question text, answers, or company context itself — every one of those lives in a file, referenced only by id.

## The flow, concretely

1. **Read the jobs file.**
2. **Skip already-applied jobs.** `tracker-agent` returns every application/posting URL already in the tracker; the coordinator removes any matching entry before doing anything else.
3. **Let the user pick** which of the remaining entries to apply to (or use `--all` / explicit selectors).
4. **Mint + seed.** For the job about to be applied to: mint a fresh `id`, write `jobs/cache/<id>/posting.md`.
5. **Prepare materials.** `identity-agent` reads the cached posting, always writes a cover letter, and only tailors a resume if `identity/documents.md` says the user opted into per-job tailoring during `/nemo:init` — otherwise it copies the base resume in unchanged. Both saved to `jobs/applied/<id>/`.
6. **Extract questions.** `browser-agent` opens the application, caches company context, fills what it can itself, writes every remaining question to `jobs/cache/<id>/questions.md`, and reports just a count.
7. **Answer questions.** If there were any, `identity-agent` answers all of them in one pass, writing directly into `questions.md`.
8. **Handle missing info.** If any answer is flagged `[NEEDS INPUT: ...]`, the coordinator stops and asks the user for just that — it doesn't let anything downstream guess.
9. **Fill, upload, submit.** `browser-agent` fills every field from `questions.md`, uploads the documents, and submits — directly, without a pre-submit summary or confirmation step.
10. **Application record.** `memory-agent` writes `jobs/applied/<id>/application-record.md`, compiled from the cache files and the materials.
11. **Update the tracker, right away.** So the tracker (and the next run's already-applied filter) stays accurate even if the run stops partway through. `browser-agent` closes the tab(s).

This repeats **per job, sequentially.**

## Browser strategy

Try the built-in browser tooling first. If a site can't be accessed that way, **stop and ask the user for explicit permission** before switching to the Chrome Connector (`skills/chrome-connector/SKILL.md`) — decided once per site, never silently. **No agent in this flow ever uses full computer/desktop-control tools — only browser-scoped tools.**

## Safety and failure handling

- `browser-agent` never fills or submits a field it — or `identity-agent` — didn't have a real answer for; a `[NEEDS INPUT: ...]` flag always stops the run for that job rather than guessing.
- If submission fails (validation error, site error, CAPTCHA loop), stop, report the exact failure, and leave the tracker untouched for that job.
- There is no pre-submit summary or confirmation gate — submission happens as soon as every question has a real answer. Everything about the application (posting, company context, every question and answer, files uploaded) is preserved in `jobs/cache/<id>/` and `jobs/applied/<id>/application-record.md` for after-the-fact review.

## Human voice, always

Everything that reaches the live form — resume (if tailored), cover letter, every answer — is written strictly in the user's own first-person voice, as the candidate, in plain human grammar. Nothing may disclose, hint at, or reference AI involvement. Only `identity-agent` produces this content. See `skills/document-generation/SKILL.md`.

## Model routing
Haiku (`application-coordinator-agent`, `browser-agent`, `memory-agent`): sequencing, id minting, cache seeding, navigation, extraction, filling, uploads, submit, file I/O — zero content generation. Sonnet (`identity-agent`): every piece of written content, produced in batches of one dispatch per job stage, not per question.
