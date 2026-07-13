---
description: Discover relevant job postings from user-specified sites, keywords, roles, and locations
argument-hint: "[sites...] [--keywords <k>] [--location <loc>]"
allowed-tools: Task, Read, Write, Glob
---

# /nemohire:source — Find Jobs

Delegate to `job-source-agent`. This is entirely optional — if you already have job postings from elsewhere, skip straight to `/nemohire:apply --jobs-file <path>` with any file that roughly looks like a list of jobs.

## Behavior

1. **Require identity.** Actually call `Read`/`Glob` against `./.claude/nemohire/identity/profile.md` — relative to the current project's working directory, **not** `~/.claude` — now, never assume its state from earlier in the conversation. If missing or still placeholder content, point to `/nemohire:init`.
2. **Collect inputs.** Use arguments if given; otherwise ask for target sites/boards, keywords, roles, and locations — falling back to `identity/job-preferences.md` and `identity/location-preferences.md` for defaults.
3. **Dispatch `job-source-agent`** with those inputs. It handles browsing (see `skills/browser-navigation/SKILL.md` for the tool order), extraction, deduplication, and writes `jobs/sourced.json` itself.
4. **Report** the short count summary it returns: new postings, duplicates skipped, sites attempted, sites that needed the Chrome Connector.

## Model

Runs on whatever model is selected for the current session.
