---
description: Apply to sourced jobs one at a time — tailors materials and answers questions live via a turn-based browser loop, then submits
argument-hint: "[job-ids...] | --all-sourced"
allowed-tools: Task, Read, Write, Glob
---

# /nemo:apply — Apply to Jobs

Orchestrated end-to-end by `application-coordinator-agent` (Haiku) — a pure relay that never writes or answers anything itself. **Applications are processed strictly sequentially — never in parallel.** There's no separate ranking or prep step: jobs go straight from `jobs/sourced/jobs.md` to a live application.

Two other agents do all the actual work, and they never talk to each other directly — only through the coordinator:
- **`browser-agent`** (Haiku, browser-scoped tools only — never full computer control): navigates, fills anything it can already answer itself from `identity/profile.md`/`identity/documents.md`, and surfaces every real question upward.
- **`identity-agent`** (Sonnet): the only agent that writes anything a human reads — it tailors the resume, writes the cover letter, and answers every live question, strictly in the user's own first-person voice, as the person, in plain human grammar, with zero AI disclosure.

## The loop, concretely

1. **Prepare.** For the selected job, `identity-agent` tailors a resume and writes a cover letter grounded in the job posting text from `jobs/sourced/jobs.md`, saving them into `jobs/applied/<company>-<role>/`.
2. **Turn 0 — open + company context.** `browser-agent` opens the application URL, pulls the company/product description, fills anything it can answer itself, and returns the first field it can't. It then stops.
3. **Relay.** The coordinator forwards that exact field/question to `identity-agent`, with the company context and job posting as grounding. It does not decide the answer itself, ever.
4. **Turn N — fill + fetch next.** The coordinator sends `browser-agent` identity-agent's exact answer; `browser-agent` fills it in verbatim, keeps filling anything else it can answer itself, and returns the next real question — or reports none remain.
5. **Repeat** steps 3–4 until nothing's left.
6. **Uploads.** `browser-agent` attaches the resume, cover letter, and any portfolio files.
7. **Submission summary.** Shown before anything is submitted: company, role, every question asked and the exact answer given, every file uploaded, any flagged/ambiguous field.
8. **Submit turn.** Only after the summary is shown.
9. **Application record.** `memory-agent` writes `jobs/applied/<company>-<role>/application-record.md` — the resume/cover-letter content submitted, every question and answer, the company context, files uploaded, timestamp/URL.
10. **Tracker + cleanup.** `tracker-agent` sets status "Applied"; `browser-agent` closes the tab(s).

This repeats **per job, sequentially.**

## Browser strategy

Try the built-in browser tooling first. If a site can't be accessed that way, **stop and ask the user for explicit permission** before switching to the Chrome Connector (`skills/chrome-connector/SKILL.md`) — decided once per site, never silently, and never per turn. **No agent in this flow ever uses full computer/desktop-control tools — only browser-scoped tools.**

## Safety and failure handling

- `browser-agent` is never given a field to fill without an answer already in hand — if information is missing from `identity/` and the job posting, the coordinator pauses and asks the user instead of guessing.
- If submission fails (validation error, site error, CAPTCHA loop), stop, report the exact failure, and leave the job's tracker status as "To Apply."
- The submission summary can never be skipped, even in a fully unattended run.

## Human voice, always

Everything that reaches the live form — resume, cover letter, every answer — is written strictly in the user's own first-person voice, as the candidate, in plain human grammar. Nothing may disclose, hint at, or reference AI involvement, and nothing should read like a template or break character. Only `identity-agent` produces this content; the coordinator and `browser-agent` never do. See `skills/document-generation/SKILL.md`.

## Model routing
Haiku (`application-coordinator-agent`, `browser-agent`): sequencing, navigation, mechanical field-filling, uploads, submit — zero content generation. Sonnet (`identity-agent`): every piece of written content and every judgment call, full stop.
