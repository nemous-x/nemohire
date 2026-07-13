---
description: Build or update your NemoHire professional identity (profile, experience, goals, salary, writing style, interview library)
argument-hint: "[--update <section>]"
allowed-tools: Read, Write, Edit, Glob, mcp__gmail__*
---

# /nemohire:init — Build Professional Identity

Run this flow directly, in this session — it's an interactive interview, so there's no reason to dispatch it to a subagent. This command must complete before any other NemoHire command produces high-quality output — `/nemohire:apply`'s cover letters and live answers all read from `./.claude/nemohire/identity/`.

## Behavior

1. **Check existing state.** Actually call `Glob`/`Read` against `./.claude/nemohire/identity/` — relative to the current project's working directory, **not** `~/.claude` — now, never assume it's already set up (or already missing) from earlier in the conversation. If files already exist and no `--update <section>` argument was given, summarize what's already known and ask whether to do a full rebuild or a targeted update. Never silently overwrite populated identity files.
2. **Look for an existing resume in the project before asking about one.** Search the working project (not just `./.claude/nemohire/`) for a plausible resume file — common names/locations like `resume.*`, `Resume.*`, `CV.*`, files under a `resume/`, `resources/`, `docs/`, or project-root location, in `.pdf`, `.docx`, or `.md`. If exactly one clear candidate is found, read it, tell the user what was found ("Found `resume.pdf` in the project — using this as your base resume unless you say otherwise"), and use it to pre-fill `identity/documents.md`'s Location/path and Format, plus extract as much of Professional/experience/achievements/skills as the resume itself contains — don't re-ask for facts the resume already answers. If multiple candidates are found, or none, ask the user directly rather than guessing which one is current.
3. **Structured interview.** Ask the user, in clearly grouped batches (not one question at a time), for whatever the resume (if found) didn't already answer:
   - Personal: name, contact info, location, timezone, portfolio/LinkedIn/GitHub links
   - Professional: current role, years of experience, industries, technologies, notable projects, achievements, leadership experience
   - Career goals: target roles, target industries, preferred companies, startup vs. enterprise, remote/hybrid/onsite
   - Salary: currency, minimum, target, ideal, negotiable (yes/no), notes. Explicitly ask: "Should salary expectations be a fixed range for every job, or flexible — matching whatever the job posting states and settling on the midpoint?" Store the answer as a rule, not just a number.
   - Work authorization: countries authorized to work in, visa sponsorship needs
   - Resume handling: ask explicitly — "Should your resume be tailored per job during `/nemohire:apply`, or should NemoHire always use your base resume as-is?" Store this as a rule in `identity/documents.md` (`Tailor per job: yes/no`). Default to **no** (use the base resume unchanged) if the user doesn't express a preference.
   - Gmail connector: ask explicitly — "Some application forms email a verification code after you submit, before the application actually goes through. Want to connect the Gmail connector so applying can retrieve that code automatically, instead of stopping to ask you for it?" Optional; if declined or unavailable, note it as not connected — that job is simply flagged as needing that input later, never guessed or skipped. Record the answer in `config.md`'s Gmail connector section.
4. **Communication analysis.** If the user pastes writing samples (emails, LinkedIn posts, past cover letters) or answers a few short prompts, analyze tone, vocabulary level, confidence, sentence rhythm, and preferred style. If no samples are given, infer style conservatively from the interview answers (and the resume's tone, if found) and flag it as a low-confidence default the user can correct later.
5. **Interview library.** Draft reusable first-person answers for: tell me about yourself, why this company, why this role, strengths, weaknesses, a leadership example, a conflict example, and the biggest achievement. Mark these as drafts — `apply-agent` refines them per-job, live, during `/nemohire:apply`.
6. **Write files.** Populate every file under `./.claude/nemohire/identity/` (see `templates/identity/` for the exact schema of each file: profile.md, experience.md, achievements.md, skills.md, communication-style.md, writing-style.md, salary-preferences.md, job-preferences.md, company-preferences.md, location-preferences.md, work-authorisation.md, documents.md, interview-library.md, memory.md). Also create `./.claude/nemohire/config.md` from `templates/tracker/config.md` if it does not exist.
7. **Confirm.** Show a short summary of what was captured and where it lives, including which resume file (if any) was found and used automatically. Do not dump full file contents back at the user unless asked.

## Notes
- This command is idempotent: re-running it with `--update <section>` (e.g. `--update salary-preferences`) only touches that one file.
- Never fabricate experience or achievements the user didn't provide. If information is missing, leave a clearly marked placeholder (`<!-- TODO: not yet provided -->`) rather than inventing detail.
- Every NemoHire agent runs on whatever model you've selected for this session — there's nothing to configure here.
