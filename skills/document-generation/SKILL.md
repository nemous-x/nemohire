---
name: document-generation
description: >
  Shared conventions for generating resumes, cover letters, and screening answers grounded in
  the user's stored identity. Use whenever resume-agent, cover-letter-agent, or screening-agent
  produce written application materials.
---

# Document Generation

## Grounding rule (non-negotiable)
Every factual claim in a generated document — a skill, a project, a metric, a title, a date — must trace back to something already present in `.claude/nemohire/identity/`. If the target job needs something not on record, do not invent it. Either omit it, or explicitly flag the gap in your output so the user can decide how to address it (add it to identity, address it in the cover letter honestly, or skip the job).

## Voice rule
Match `identity/writing-style.md` and `identity/communication-style.md` — tone, vocabulary level, sentence rhythm, confidence. A generated cover letter should be indistinguishable in voice from something the user would write themselves, adjusted only for the specific company/role content.

## Structure
- Resumes: reorder and re-emphasize existing `identity/experience.md` / `identity/achievements.md` content to mirror the target posting's language and priorities. Use `templates/resumes/tailoring-checklist.md` as a guide.
- Cover letters: use `templates/cover-letter/` as a structural starting point, not a fill-in-the-blank script. Reference specifics from the actual posting and any company research gathered.
- Screening answers: start from `identity/interview-library.md` for standard questions; adapt rather than reuse verbatim. Respect any stated length/character limits from the form.

## Quality bar
Before considering a document ready for `qa-agent` review: no leftover template placeholders, no generic filler sentences that could apply to any company, and no claims that don't trace back to identity.
