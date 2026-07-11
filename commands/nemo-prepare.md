---
description: Generate tailored application materials (resume, cover letter, screening answers, company research, interview notes) for selected ranked jobs
argument-hint: "[job-ids...] | --top <n>"
allowed-tools: Task, Read, Write, Glob
---

# /nemo:prepare — Prepare Applications

Orchestrates `resume-agent`, `cover-letter-agent`, `screening-agent`, and `qa-agent`.

## Behavior

1. **Select jobs.** From `.claude/nemohire/jobs/ranked/ranking.md`, use the jobs the user specifies, or the top N if `--top <n>` is given, or ask which ones if neither is provided.
2. **Per job**, in a `jobs/prepared/<company>-<role>/` folder:
   - `resume-agent` tailors the base resume (from `identity/documents.md`) to the posting's requirements — reordering and emphasizing, never fabricating experience.
   - `cover-letter-agent` drafts a cover letter using `identity/writing-style.md` and `templates/cover-letter/`.
   - `screening-agent` drafts answers to the posting's actual application questions plus standard ones, reusing and adapting `identity/interview-library.md`.
   - Research the company briefly (mission, recent news, size) and save it as `company-research.md`.
   - Compile interview prep notes (`interview-notes.md`) anticipating likely questions from the job description.
   - `qa-agent` reviews every generated file for factual consistency with `identity/` (no invented facts), tone consistency with `writing-style.md`, and basic quality (no placeholder text left in).
3. **Save** all outputs under `.claude/nemohire/jobs/prepared/<company>-<role>/`.
4. **Report** which jobs were prepared and flag anything QA couldn't verify.

## Model routing
Sonnet for resume tailoring, cover letters, screening answers, and company research (writing/reasoning). Haiku for file assembly and QA's mechanical checks (placeholder scan, structure check); Sonnet for QA's factual/tone judgment calls.
