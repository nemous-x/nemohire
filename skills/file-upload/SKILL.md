---
name: file-upload
description: >
  Detects file upload fields on job application forms and attaches the correct document
  (resume, cover letter, portfolio file) to each. Used by browser-agent directly during the
  upload turn of /nemo:apply — there's no separate upload agent.
---

# File Upload

## Steps

1. Identify every upload-type field on the current page, its label, and accepted file types (PDF, DOCX, etc. — read `accept` attributes or visible instructions if available).
2. Match fields to files by purpose, not just by order on the page:
   - Resume/CV field → `jobs/applied/<company>-<role>/resume.*` (written there by `identity-agent` at the start of this job's apply attempt)
   - Cover letter field → `jobs/applied/<company>-<role>/cover-letter.*`
   - Portfolio/work-samples field → entries listed under `identity/documents.md`
3. If the field's accepted format doesn't match what's available (e.g. field wants `.pdf` but only a `.md` version exists), flag this rather than uploading a mismatched or broken file. Note it in the output so the calling agent can convert first or ask the user.
4. Confirm each upload succeeded (most ATS forms show a filename or checkmark after upload) before moving to the next field.
5. Never upload a file to a field you're not confident about — an unclear or unlabeled upload field should be flagged for the user, not guessed at.

## Output contract
Return: field → file mapping actually performed, and any field that couldn't be filled with the reason (format mismatch, ambiguous purpose, missing source file).
