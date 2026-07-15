---
name: document-generation
description: >
  Shared conventions for generating resumes, cover letters, and live application answers,
  grounded in the user's stored identity. This is apply-agent's rulebook — it's the only agent
  that produces this content, as one part of applying to a single job in one dispatch.
---

# Document Generation

## Grounding rule (non-negotiable)
Every factual claim — a skill, a project, a metric, a title, a date — must trace back to something already present in `./.claude/nemohire/identity/`. If the job needs something not on record, do not invent it. Either omit it or flag the gap so the user can decide (add it to identity, address it honestly, or skip the job).

## Voice rule
Match `identity/writing-style.md` and `identity/communication-style.md` — tone, vocabulary, sentence rhythm, confidence. A cover letter or answer should be indistinguishable in voice from something the user would write themselves, adjusted only for the specific company/role.

## Human-voice rule (non-negotiable)
Every piece of content that goes into an application — cover letter, resume, live-form answer, even a one-word answer — is written in the user's own first-person voice, in plain human grammar. It must never:
- disclose, hint at, or reference AI assistance, generation, or drafting
- break character with disclaimers, hedges, or meta-commentary about how it was produced
- read like a template or like something written *about* the candidate rather than *by* them

`apply-agent` is the only agent that produces this content, and this rule applies to every word it writes.

## Structure
- **Resume — opt-in only.** Check `identity/documents.md`'s "Tailor per job" setting. If off (the default), don't touch the resume and don't write a per-job copy — upload the base resume directly. If on, reorder and re-emphasize existing `identity/experience.md`/`identity/achievements.md` content to mirror this posting's language, and write it to `./.claude/nemohire/jobs/resumes/<seq>-<id>.<ext>`. Use `templates/resumes/tailoring-checklist.md` as a guide.
- **Cover letter — always written, per job.** Reference specifics from the posting, never generic filler. Use `templates/cover-letter/` as a structural starting point, not a fill-in-the-blank script. Composed in memory and written into the job's details file once, in the Cover letter section — not a separate file.
- **Answers — composed in the moment, as you encounter each field.** Start from `identity/interview-library.md` for standard questions; adapt rather than reuse verbatim. There's no separate questions file to hand off to another agent — you read the form, you know the identity, you write the answer, you fill the field, all in the same pass.

## Reading context
There's no pre-written posting file to read — nothing under `jobs/details/` exists for this job yet. Read the posting straight off the live page once you've opened `application_url` (see `skills/browser-navigation/SKILL.md`); that's your only source, fresh for this job, don't assume anything about it from an earlier job in the same batch.

## When you can't ground an answer
Don't fabricate it. Flag exactly what's missing (this is what `needs_input` means for that job) and keep going with everything else you can complete — only stop the whole job if the missing piece is required to submit at all.

## Quality bar
Before submitting: no leftover template placeholders, no generic filler that could apply to any company, no claim that doesn't trace back to identity. Self-check against this bar and the human-voice rule before filling the final field — there's no separate QA pass.
