---
name: email-sync
description: >
  Reads hiring-related messages from the connected mail source since the last sync point and
  classifies them into rejection / interview / recruiter response / assessment / offer / other.
  Use for /nemo:sync-email via email-agent.
---

# Email Sync

## Steps

1. Read `.claude/nemohire/emails/last-sync.json` for the last sync timestamp (ISO-8601). If absent, default to 30 days back and note that this is a first sync.
2. Search the connected mail source for messages after that timestamp. Prefer a targeted search (hiring-related keywords, known ATS sender domains like greenhouse.io, lever.co, workday.com, myworkdayjobs.com) over pulling the entire inbox.
3. Classify each candidate message into exactly one bucket:
   - **rejection** — explicit "not moving forward" language
   - **interview** — invitation to schedule or confirmation of an interview
   - **recruiter response** — a recruiter reaching out or replying, not yet an interview invite
   - **assessment** — take-home test, coding challenge, or assessment link
   - **offer** — an actual offer of employment
   - **other** — not hiring-related or doesn't fit the above; discard without further reporting
4. For each non-"other" message, produce a short structured note (company, role if known, classification, one-line summary) and hand it to `tracker-agent` to update the corresponding row.
5. If a message is genuinely ambiguous between two categories, escalate just that single message for a Sonnet-level judgment call — don't guess and don't escalate everything by default.
6. Update `last-sync.json` with the new timestamp only after the sync completes successfully.

## Output contract
Return only aggregate counts (X interviews, X rejections, X recruiter responses, X assessments, X pending actions) — never dump raw email bodies into the conversation.
