---
description: Apply to prepared jobs one at a time via a turn-based browser loop, uploading documents and submitting
argument-hint: "[job-ids...] | --all-prepared"
allowed-tools: Task, Read, Write, Glob
---

# /nemo:apply — Apply to Jobs

Orchestrated end-to-end by `application-coordinator-agent` (Sonnet). **Applications are processed strictly sequentially — never in parallel.** Within each application, `browser-agent` (Haiku) and the coordinator run a **turn-based loop**: browser-agent never drafts an answer and the coordinator never touches the browser — they hand a single field/question back and forth, one at a time, until the form is done.

## The loop, concretely

1. **Turn 0 — open + company context.** `browser-agent` opens the application URL, pulls the company/product description from the posting or an About section, and returns it along with the first unanswered field/question. It then stops.
2. **Coordinator answers.** The coordinator reads the company context (grounding "why this company" style answers in something real, not filler) and produces an answer for that one field — reusing `jobs/prepared/<company>-<role>/` where possible, or dispatching `screening-agent`/`cover-letter-agent` for anything new, using the company context just learned.
3. **Turn N — fill + fetch next.** The coordinator sends browser-agent the exact answer text; browser-agent fills that field verbatim, then looks for and returns the next unanswered field — or reports that none remain.
4. **Repeat** steps 2–3 until browser-agent reports no more question fields.
5. **Uploads.** `upload-agent` attaches resume, cover letter, and portfolio files to the detected upload fields (per `skills/file-upload/SKILL.md`).
6. **Submission summary.** Shown to the user before anything is submitted: company, role, every question asked and the exact answer given, every file uploaded, and any flagged/ambiguous field.
7. **Submit turn.** Only after the summary is shown, the coordinator dispatches browser-agent's dedicated submit turn.
8. **Tracker + cleanup.** `tracker-agent` sets status "Applied" with the date (ISO-8601) and job posting URL; browser-agent closes the tab(s); the job folder moves from `prepared/` to `applied/`.

This repeats **per job, sequentially** — the whole loop above runs to completion (or a stop) for one job before the next job starts.

## Browser strategy

Try the built-in browser tooling first. If a site can't be accessed that way, **stop and ask the user for explicit permission** before switching to the Chrome Connector (`skills/chrome-connector/SKILL.md`) — this decision is never made silently, and it's made once per site, not once per turn.

## Safety and failure handling

- browser-agent is never given a field to fill without an answer already in hand — if information is missing from `identity/` and `jobs/prepared/`, the coordinator pauses and asks the user instead of guessing.
- If submission fails (validation error, site error, CAPTCHA loop), stop, report the exact failure, and leave the job's tracker status as "To Apply" — do not mark it Applied unless submission actually succeeded.
- The submission summary step can never be skipped, even in a fully unattended run.

## Model routing
Haiku (`browser-agent`): navigation, one field/question extraction per turn, filling in the answer it's given, upload attachment, submit click. Sonnet (`application-coordinator-agent`, `screening-agent`, `cover-letter-agent`): reading company context, producing every answer, and all judgment calls. Haiku is never asked to invent an answer, and Sonnet never touches the browser directly.
