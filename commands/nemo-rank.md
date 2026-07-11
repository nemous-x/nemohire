---
description: Rank sourced jobs against your identity, skills, salary and preferences
allowed-tools: Task, Read, Write, Glob
---

# /nemo:rank — Rank Jobs

Delegate to the `job-ranking-agent`.

## Behavior

1. **Inputs.** Read every unranked posting in `.claude/nemohire/jobs/sourced/jobs.md` and the full `identity/` directory.
2. **Score each job** on: identity/skill match, salary match (respecting the fixed-vs-flexible rule from `identity/salary-preferences.md`), company preference fit, location/remote fit, and career growth trajectory. Use a simple 0–100 composite score with a one-line rationale per dimension — keep it scannable, not a wall of text.
3. **Write output** to `.claude/nemohire/jobs/ranked/ranking.md`, sorted highest score first, with the rationale and a link back to the full posting in `jobs/sourced/jobs.md`.
4. **Report** the top 5 matches inline as a short list, and note the total count ranked.

## Model routing
Sonnet — this step is comparative reasoning against the user's identity, not extraction.
