---
description: Apply to sourced (or externally supplied) jobs one at a time — tailors materials and answers questions live via a turn-based browser loop, then submits
argument-hint: "[job-ids...] | --all-sourced | --jobs-file <path>"
allowed-tools: Task, Read, Write, Glob
---

# /nemo:apply — Apply to Jobs

Orchestrated end-to-end by `application-coordinator-agent` (Haiku) — a pure relay that never writes, answers, or holds large text itself. **Applications are processed strictly sequentially — never in parallel.** There's no separate ranking or prep step: jobs go straight from `.claude/nemohire/jobs/jobs.json` to a live application.

## You don't need to have run `/nemo:source`

If you already have job postings from somewhere else, pass `--jobs-file <path>` pointing at a JSON file (see `templates/tracker/jobs-schema.md` for the shape). `memory-agent` maps it onto the `jobs.json` schema, computes an id for anything missing one, marks it `source: "manual"`, and merges it in before the run starts — no need to go through `/nemo:source` at all.

## Every job has an id — and that id is the only thing that moves between agents

The moment a job is in `jobs.json`, it has a stable `id`. That id is the key efficiency mechanism in this flow: large text (the job posting, the company/product context) is written once and read by id — it is never re-transmitted between agents on every turn.

- **`browser-agent`** (Haiku, browser-scoped tools only — never full computer control): looks up the posting by id, navigates, writes the company/product description it extracts straight to `jobs/cache/<id>/company-context.md` itself, fills anything it can already answer itself from `identity/profile.md`/`identity/documents.md`, and surfaces every real question upward as just the question text.
- **`identity-agent`** (Sonnet): the only agent that writes anything a human reads. It receives just `{id, question}` (or `{id, "prepare materials"}`) — nothing more — and reads the posting and cached company context itself, by id, before answering or writing. Always strictly in the user's own first-person voice, as the person, in plain human grammar, with zero AI disclosure.
- **`application-coordinator-agent`** (Haiku): relays `{id, question}` to `identity-agent` and the resulting answer back to `browser-agent`. It never holds, inspects, or forwards the company context or posting text itself.

## The loop, concretely

1. **(Optional) Ingest external jobs.** If `--jobs-file` was given, `memory-agent` merges it into `jobs.json` first.
2. **Prepare.** For the selected job's `id`, `identity-agent` tailors a resume and writes a cover letter (reading the posting itself by id), saving them into `jobs/applied/<id>/`.
3. **Turn 0 — open + cache company context.** `browser-agent` opens the application (via `application_url` looked up by id), writes the company/product description to `jobs/cache/<id>/company-context.md` itself, fills anything it can answer itself, and returns just the first field it can't. It then stops.
4. **Relay.** The coordinator forwards `{id, question}` — nothing else — to `identity-agent`.
5. **Turn N — fill + fetch next.** The coordinator sends `browser-agent` identity-agent's exact answer; `browser-agent` fills it in verbatim, keeps filling anything else it can answer itself, and returns the next real question — or reports none remain.
6. **Repeat** steps 4–5 until nothing's left.
7. **Uploads.** `browser-agent` attaches the resume, cover letter, and any portfolio files from `jobs/applied/<id>/`.
8. **Submission summary.** Shown before anything is submitted: company, role, every question asked and the exact answer given, every file uploaded, any flagged/ambiguous field.
9. **Submit turn.** Only after the summary is shown.
10. **Application record.** `memory-agent` writes `jobs/applied/<id>/application-record.md` and sets `jobs.json`'s entry to `status: "applied"`.
11. **Tracker + cleanup.** `tracker-agent` sets status "Applied"; `browser-agent` closes the tab(s).

This repeats **per job, sequentially.**

## Browser strategy

Try the built-in browser tooling first. If a site can't be accessed that way, **stop and ask the user for explicit permission** before switching to the Chrome Connector (`skills/chrome-connector/SKILL.md`) — decided once per site, never silently, and never per turn. **No agent in this flow ever uses full computer/desktop-control tools — only browser-scoped tools.**

## Safety and failure handling

- `browser-agent` is never given a field to fill without an answer already in hand — if information is missing from `identity/` and the job posting, the coordinator pauses and asks the user instead of guessing.
- If submission fails (validation error, site error, CAPTCHA loop), stop, report the exact failure, and leave the job's `jobs.json`/tracker status untouched.
- The submission summary can never be skipped, even in a fully unattended run.

## Human voice, always

Everything that reaches the live form — resume, cover letter, every answer — is written strictly in the user's own first-person voice, as the candidate, in plain human grammar. Nothing may disclose, hint at, or reference AI involvement, and nothing should read like a template or break character. Only `identity-agent` produces this content; the coordinator and `browser-agent` never do. See `skills/document-generation/SKILL.md`.

## Model routing
Haiku (`application-coordinator-agent`, `browser-agent`, `memory-agent`): sequencing, navigation, mechanical field-filling, uploads, submit, file I/O — zero content generation, zero large-payload relaying. Sonnet (`identity-agent`): every piece of written content and every judgment call, full stop.
