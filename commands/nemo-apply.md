---
description: Apply to prepared jobs one at a time via browser automation, uploading documents and submitting
argument-hint: "[job-ids...] | --all-prepared"
allowed-tools: Task, Read, Write, Glob
---

# /nemo:apply — Apply to Jobs

Orchestrated end-to-end by `application-coordinator-agent`. **Applications are processed strictly sequentially — never in parallel** — so the user can interrupt, and so site rate limits and CAPTCHA/verification prompts are handled one at a time.

## Behavior

For each prepared job, in order:

1. Open the job's application URL using the built-in browser tooling first.
2. If the site can't be accessed with built-in tooling, **stop and ask the user for explicit permission** before switching to the Chrome Connector (`skills/chrome-connector/SKILL.md`). Never switch silently.
3. `browser-agent` (Haiku) navigates the application flow, identifies form fields, extracts any custom screening questions the site asks, and detects file upload fields.
4. `screening-agent` / `cover-letter-agent` (Sonnet) generate or adapt answers for any question not already covered in `jobs/prepared/<company>-<role>/`.
5. `upload-agent` uses `skills/file-upload/SKILL.md` to attach the tailored resume, cover letter, and portfolio files to the correct fields.
6. Before submitting, **show the user a submission summary**: company, role, every answer that will be submitted, every file being uploaded, and any field flagged as important or ambiguous (e.g. legal attestations, salary fields, start-date commitments).
7. **Submit** the application.
8. Update the tracker (`tracker-agent`) — status "Applied", date applied (ISO-8601), job posting URL.
9. **Close the browser tab(s)** used for this job once submission is confirmed, before moving to the next job.
10. Move the job's folder reference from `prepared` to `applied` in tracking state.

## Safety and failure handling

- If a site requires information not present anywhere in `identity/` or `jobs/prepared/`, pause and ask the user rather than guessing.
- If submission fails (validation error, site error, CAPTCHA loop), stop, report the exact failure, and leave the job's tracker status as "To Apply" — do not mark it Applied unless submission actually succeeded.
- Never submit without the user-visible summary step above, even in a fully unattended run — post the summary as output and proceed only after it's been shown.

## Model routing
Haiku: navigation, field detection, extraction, uploads. Sonnet: answer generation/personalization and any judgment calls flagged during navigation.
