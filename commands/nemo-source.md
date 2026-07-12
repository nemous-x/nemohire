---
description: Discover relevant job postings from user-specified sites, keywords, roles, and locations
argument-hint: "[sites...] [--keywords <k>] [--location <loc>]"
allowed-tools: Task, Read, Write, Glob
---

# /nemo:source — Find Jobs

Delegate to the `job-source-agent`. This is optional — if you already have job postings from elsewhere, skip straight to `/nemo:apply --jobs-file <path>` instead.

## Behavior

1. **Require identity.** If `.claude/nemohire/identity/profile.md` doesn't exist, tell the user to run `/nemo:init` first.
2. **Collect inputs.** Use arguments if given; otherwise ask the user for: target websites/job boards, keywords, target roles, and locations. Fall back to `identity/job-preferences.md` and `identity/location-preferences.md` for defaults.
3. **Browse.** Per the plugin's browser strategy (see `skills/browser-navigation/SKILL.md`): try the built-in browser tooling first. If a site can't be accessed, stop and ask the user for explicit permission before falling back to the Chrome Connector (`skills/chrome-connector/SKILL.md`). Never switch browsers silently.
4. **Extract, don't rewrite.** For each posting pull: title, company, location, salary (if listed), full description, requirements, posting URL, and application URL. Copy description text verbatim — do not summarize or rephrase it; `/nemo:apply` needs the original text to ground `identity-agent`'s resume tailoring, cover letter, and live answers.
5. **Assign an id.** Compute `<company-slug>-<title-slug>-<6-char-hash>` per `templates/tracker/jobs-schema.md` for every new posting.
6. **De-duplicate.** Before appending, check `.claude/nemohire/jobs/jobs.json` for an entry with the same id and skip re-adding it.
7. **Save.** Append structured entries (with `source: "nemohire"`, `status: "sourced"`) to `.claude/nemohire/jobs/jobs.json`, via `memory-agent`.
8. **Close tabs** opened during sourcing once extraction for each posting is complete.
9. **Report** a short count summary: N new postings found, N duplicates skipped, N sites attempted, N sites that needed the Chrome Connector.

## Model routing
Haiku for the entire flow — this is navigation, extraction, and formatting, not reasoning.
