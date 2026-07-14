---
description: Discover relevant job postings from user-specified sites, keywords, roles, and locations
argument-hint: "[sites...] [--keywords <k>] [--location <loc>]"
allowed-tools: Task, Read, Write, Glob, Grep
---

# /nemohire:source — Find Jobs

Delegate to `job-source-agent`. This is entirely optional — if you already have job postings from elsewhere, skip straight to `/nemohire:apply --jobs-file <path>` with any file that roughly looks like a list of jobs.

## Path resolution

Every path below is relative to **the project root — the directory that contains `.claude/`**. Never assume `.claude/nemohire/` itself is the working directory; always use the full path starting with `./.claude/nemohire/`.

## Behavior

1. **Require identity.** `Read`/`Glob` `./.claude/nemohire/identity/profile.md` — if missing or still placeholder content, point to `/nemohire:init`.
2. **Collect inputs.** Use arguments if given; otherwise ask for target sites/boards, keywords, roles, and locations — falling back to `./.claude/nemohire/identity/job-preferences.md` and `location-preferences.md` for defaults.
3. **Dispatch `job-source-agent`** with those inputs. It handles browsing (see `skills/browser-navigation/SKILL.md` for the tool order), extraction, deduplication, and appends new rows/details files to the ledger itself — see `templates/tracker/jobs-ledger-schema.md`.
4. **Report** the short count summary it returns: new postings, duplicates skipped, sites attempted, sites that needed the Chrome Connector.

## Model

Runs on whatever model is selected for the current session.
