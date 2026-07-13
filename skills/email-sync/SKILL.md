---
name: email-sync
description: >
  Reads hiring-related messages from the connected mail source since the last sync point and
  classifies them into rejection / interview / recruiter response / assessment / offer / other.
  Use for /nemohire:sync-email via email-agent.
---

# Email Sync

## Steps

1. Read `./.claude/nemohire/emails/last-sync.json` for the last sync timestamp (ISO-8601). If absent, default to 30 days back and note that this is a first sync.
2. Search the connected mail source for messages after that timestamp. Prefer a targeted search (hiring-related keywords, known ATS sender domains like greenhouse.io, lever.co, workday.com, myworkdayjobs.com) over pulling the entire inbox.
3. Classify each candidate message into exactly one bucket:
   - **rejection** — explicit "not moving forward" language
   - **interview** — invitation to schedule or confirmation of an interview
   - **recruiter response** — a recruiter reaching out or replying, not yet an interview invite
   - **assessment** — take-home test, coding challenge, or assessment link
   - **offer** — an actual offer of employment
   - **other** — not hiring-related or doesn't fit the above; discard without further reporting
4. For each non-"other" message, update the corresponding row in `tracker/applications.md` directly — no separate tracker agent involved.
5. If a message is genuinely ambiguous between two categories, use your best judgment rather than guessing wildly; if it's truly unresolvable, flag that one message for the user instead of forcing a classification.
6. Update `last-sync.json` with the new timestamp only after the sync completes successfully.

## Output contract
Return only aggregate counts (X interviews, X rejections, X recruiter responses, X assessments, X pending actions) — never dump raw email bodies into the conversation.
