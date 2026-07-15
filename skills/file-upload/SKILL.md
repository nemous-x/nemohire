---
name: file-upload
description: >
  Attaches the correct document (resume, cover letter, portfolio file) to each file-upload field
  on a job application form. apply-agent only carries Playwright MCP by default, which attaches
  files directly against the input element — no OS dialog ever appears. The resume is always
  attached when any resume/CV field exists, whether or not the field says it's required. Used by
  apply-agent directly during the fill step — there's no separate upload agent.
---

# File Upload

## The resume is always attached — never conditional on "required"

If the form has a resume/CV/attachment field at all, a resume gets attached to it before submitting, every time — regardless of whether the field is marked required, optional, or unlabeled either way. Skipping the resume because a field wasn't flagged mandatory is a real failure, not an acceptable shortcut: the whole point of the application is the resume reaching the employer. Treat this as a submit-blocking requirement, same tier as a genuinely required text field.

## Playwright's native file-attach — the only method apply-agent needs

Playwright can attach a file directly to a file input element without ever opening an operating system dialog:

1. Identify the upload field from the page's structured read (label, accepted formats via its `accept` attribute, and which element it is).
2. Use Playwright's native file-attach action, pointed at that element, with the real file path on disk. This injects the file directly — no OS picker appears, and none should. If one does appear, that's a sign something's wrong with how the field was targeted, not a normal alternate path to work around.
3. Confirm the attach succeeded from the page's own state (most ATS forms show a filename or checkmark) — check this as part of the same read, not a separate step.

apply-agent doesn't carry the Chrome Connector by default (see `agents/apply-agent.md`), so there's no second method to choose between in the normal case. If a job genuinely can't be completed without it, that's a `needs_input` for the user to approve — not something to work around here.

## Matching fields to files

`<seq>-<id>` is minted for this job (see `templates/tracker/jobs-ledger-schema.md`):
- Resume/CV field → `./.claude/nemohire/jobs/resumes/<seq>-<id>.<ext>` **if it exists**. A per-job resume file only exists when the user opted into per-job tailoring (`identity/documents.md`'s "Tailor per job: yes"). If tailoring is off (the default), no per-job copy is made at all — upload the base resume directly from the path recorded in `identity/documents.md`'s "Location/path" instead. Check for the per-job file first; fall back to the base path only if it's genuinely absent, never guess which one is "more current."
- Cover letter field → typed directly into the form field, composed in memory as part of the job (see `agents/apply-agent.md`), not read from a separate file.
- Portfolio/work-samples field → entries listed under `identity/documents.md`

## When something doesn't fit

If the field's accepted format doesn't match what's available (e.g. it wants `.pdf` but only a `.md` version exists), flag this rather than uploading a mismatched or broken file. Never upload to a field you're not confident about — an unclear or unlabeled upload field is flagged for the user, not guessed at.

## Output contract
Return: field → file mapping actually performed, and any field that couldn't be filled with the reason (format mismatch, ambiguous purpose, missing source file, upload not confirmed).
