---
description: Build or update your NemoHire professional identity (profile, experience, goals, salary, writing style, interview library)
argument-hint: "[--update <section>]"
allowed-tools: Read, Write, Edit, Glob, Task
---

# /nemo:init — Build Professional Identity

Invoke the `identity-agent` (see `agents/identity-agent.md`) to run this flow. This command must complete before any other NemoHire command produces high-quality output — `/nemo:apply`'s resume tailoring, cover letters, and live answers all read from `.claude/nemohire/identity/`.

## Behavior

1. **Check existing state.** Look for `.claude/nemohire/identity/`. If files already exist and no `--update <section>` argument was given, summarize what's already known and ask whether to do a full rebuild or a targeted update. Never silently overwrite populated identity files.
2. **Structured interview.** Ask the user, in clearly grouped batches (not one question at a time), for:
   - Personal: name, contact info, location, timezone, portfolio/LinkedIn/GitHub links
   - Professional: current role, years of experience, industries, technologies, notable projects, achievements, leadership experience
   - Career goals: target roles, target industries, preferred companies, startup vs. enterprise, remote/hybrid/onsite
   - Salary: currency, minimum, target, ideal, negotiable (yes/no), notes. Explicitly ask: "Should salary expectations be a fixed range for every job, or flexible — matching whatever the job posting states and settling on the midpoint?" Store the answer as a rule, not just a number.
   - Work authorization: countries authorized to work in, visa sponsorship needs
   - Resume handling: ask explicitly — "Should your resume be tailored per job during `/nemo:apply`, or should NemoHire always use your base resume as-is?" Store this as a rule in `identity/documents.md` (`Tailor per job: yes/no`). Default to **no** (use the base resume unchanged) if the user doesn't express a preference — tailoring only happens when asked for.
3. **Communication analysis.** If the user pastes writing samples (emails, LinkedIn posts, past cover letters) or answers a few short prompts, analyze tone, vocabulary level, confidence, sentence rhythm, and preferred style. If no samples are given, infer style conservatively from the interview answers and flag it as a low-confidence default the user can correct later.
4. **Interview library.** Draft reusable first-person answers for: tell me about yourself, why this company, why this role, strengths, weaknesses, a leadership example, a conflict example, and the biggest achievement. Mark these as drafts — `identity-agent` refines them per-job, live, during `/nemo:apply`.
5. **Write files.** Populate every file under `.claude/nemohire/identity/` (see `templates/identity/` for the exact schema of each file: profile.md, experience.md, achievements.md, skills.md, communication-style.md, writing-style.md, salary-preferences.md, job-preferences.md, company-preferences.md, location-preferences.md, work-authorisation.md, documents.md, interview-library.md, memory.md). Also create `.claude/nemohire/config.md` from `templates/tracker/config.md` if it does not exist.
6. **Confirm.** Show a short summary of what was captured and where it lives. Do not dump full file contents back at the user unless asked.

## Model routing
Use Sonnet for the interview, style analysis, and interview-library drafting (this is reasoning/writing work). Use Haiku only for mechanical file writes if delegated to `memory-agent`.

## Notes
- This command is idempotent: re-running it with `--update <section>` (e.g. `--update salary-preferences`) only touches that one file.
- Never fabricate experience or achievements the user didn't provide. If information is missing, leave a clearly marked placeholder (`<!-- TODO: not yet provided -->`) rather than inventing detail.
