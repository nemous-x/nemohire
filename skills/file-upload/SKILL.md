---
name: file-upload
description: >
  Attaches the correct document (resume, cover letter, portfolio file) to each file-upload field
  on a job application form. The method depends on which browser tool is active: Playwright MCP
  attaches files directly against the input element, no OS dialog involved. The Chrome Connector
  doesn't have that capability, so when it's active, attaching a file means letting the native OS
  file picker open and using desktop control to select the file in it, then releasing control back
  to the browser. The resume is always attached when any resume/CV field exists, whether or not
  the field says it's required. Used by apply-agent directly during the fill step of
  /nemohire:apply — there's no separate upload agent.
---

# File Upload

## The resume is always attached — never conditional on "required"

If the form has a resume/CV/attachment field at all, a resume gets attached to it before submitting, every time — regardless of whether the field is marked required, optional, or unlabeled either way. Skipping the resume because a field wasn't flagged mandatory is a real failure, not an acceptable shortcut: the whole point of the application is the resume reaching the employer. Treat this as a submit-blocking requirement, same tier as a genuinely required text field.

## Two methods, one per active browser tool — never mix them up

### Playwright MCP (the default)

Playwright can attach a file directly to a file input element without ever opening an operating system dialog. This is the default, expected path:

1. Identify the upload field from the page's structured read (label, accepted formats via its `accept` attribute, and which element it is).
2. Use Playwright's native file-attach action, pointed at that element, with the real file path on disk. This injects the file directly — no OS picker appears, and none should.
3. Confirm the attach succeeded from the page's own state (most ATS forms show a filename or checkmark) — check this as part of the same read, not a separate step.

### Chrome Connector (only when that's the active tool)

The Chrome Connector doesn't expose Playwright's native file-attach capability — clicking an upload button here genuinely opens the operating system's own file picker (Finder or equivalent), and that's expected and correct in this path, not a mistake to avoid:

1. Click the upload button/field in the browser as normal.
2. When the native OS file dialog opens, switch to desktop control to operate it directly: navigate to the file's location (or use the dialog's path/search field), select the correct file, and confirm (Open/Choose).
3. Once the dialog closes and the browser shows the file attached, release control back to the Chrome Connector's browser tools and continue from there — don't keep driving the desktop once the file is selected.
4. Confirm the attach succeeded the same way — a filename or checkmark visible on the page.

Don't apply the Playwright method while the Chrome Connector is active, or vice versa — check which tool is actually driving the current job before choosing how to upload.

## Matching fields to files

`<id>` is the id minted for this job (see `templates/tracker/jobs-ledger-schema.md`):
- Resume/CV field → `./.claude/nemohire/jobs/resumes/<id>.<ext>` **if it exists**. A per-job resume file only exists when the user opted into per-job tailoring (`identity/documents.md`'s "Tailor per job: yes"). If tailoring is off (the default), no per-job copy is made at all — upload the base resume directly from the path recorded in `identity/documents.md`'s "Location/path" instead. Check for the per-job file first; fall back to the base path only if it's genuinely absent, never guess which one is "more current."
- Cover letter field → typed directly into the form field, composed in memory as part of the job (see `agents/apply-agent.md`), not read from a separate file.
- Portfolio/work-samples field → entries listed under `identity/documents.md`

## When something doesn't fit

If the field's accepted format doesn't match what's available (e.g. it wants `.pdf` but only a `.md` version exists), flag this rather than uploading a mismatched or broken file. Never upload to a field you're not confident about — an unclear or unlabeled upload field is flagged for the user, not guessed at.

## Output contract
Return: field → file mapping actually performed, and any field that couldn't be filled with the reason (format mismatch, ambiguous purpose, missing source file, upload not confirmed).
