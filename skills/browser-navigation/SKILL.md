---
name: browser-navigation
description: >
  Use whenever NemoHire needs to open a job board or application page, extract a posting, or fill
  and submit a form. Playwright MCP is the default browsing tool for every job, including logging
  in when a site asks for it — that happens right in the same Playwright browser window, not by
  switching tools. The Chrome Connector is a reactive fallback only, reserved for a page that
  genuinely won't load in Playwright at all, or a login that genuinely can't be done through
  Playwright's session. There's no fixed list of which sites need which tool; that's decided per
  job, from what's actually encountered on the page.
---

# Browser Navigation

## Browsing tool order

1. **Playwright MCP** — the default, first choice, for every job-search browsing task, including logging in when a site asks for it.
2. **Chrome Connector (`skills/chrome-connector/SKILL.md`)** — reactive only, never proactive, never the default reaction to a login wall, never based on a hardcoded list of site names. Reserved for a few specific cases:
   - The page genuinely won't load in Playwright at all (timeout, blocked, broken rendering).
   - Logging in through Playwright genuinely isn't workable — the session isn't one the user can interact with directly, or the site specifically needs the user's own already-authenticated browser profile rather than a fresh login.
   Always per-site, always with the user's explicit go-ahead, never silent, never decided in advance from a list of "known" login-gated boards.

**Whether "switch to the Chrome Connector" is even an option depends on which agent hit the wall.** `job-source-agent` carries the Chrome Connector in its own toolset, so for sourcing, it genuinely does switch to it in these two cases. `apply-agent` does not carry it by default (see `agents/apply-agent.md`) — for applying, hitting either case means stopping the job as `needs_input` with a clear note ("site needs the Chrome Connector — approve enabling it"), not silently switching tools. Either way, the trigger conditions are identical; only what happens next differs by agent.

**A login wall by itself is not a reason to reach for the Chrome Connector.** Playwright's browser is a normal, real browser window — when a site asks the user to log in, ask them to do it right there, in the same Playwright session that's already open on the right page, then continue once they confirm. This is the default for every login; the Chrome Connector is the exception, not the destination login requests route to automatically.

Whichever tool is active for a given job, the steps and discipline below apply the same way.

## Remember which sites Playwright can't load

The first time Playwright genuinely fails to load a domain (not a one-off network blip — a real timeout, block, or broken rendering), record that domain in `./.claude/nemohire/browser-fallback-sites.json` (a plain array of hostnames) before switching tools or stopping. On every subsequent job, check this list first: if a job's domain is already on it, `job-source-agent` goes straight to the Chrome Connector; `apply-agent` goes straight to `needs_input` without wasting a doomed Playwright attempt first. This file is small, per-project, learned data, not plugin configuration — it just grows as real jobs are processed. It is never pre-populated with site names; it only ever contains domains this project has actually seen fail.

## One session, used by one dispatch at a time — and one job at a time within it

There is one browser session behind every job in a batch — one set of tabs, one navigation state. Two dispatches touching it at once, even for two unrelated jobs, will step on each other's tabs and corrupt both jobs' results. `/nemohire:apply`/`/nemohire:continue` always send a whole batch (`--batch-size`, default 5) to `apply-agent` in a single dispatch, wait for it to fully return, then send the next batch — never two dispatches in flight together. Inside that one dispatch, `apply-agent` itself works through its batch strictly one job at a time — finishing a job's outcome and details file completely before opening the next job's URL in the same session — so the "one dispatch at a time" rule and the "one job at a time" rule are really the same rule, just applied at two levels.

## Steps

1. Check `browser-fallback-sites.json` for this job's domain — if it's already there, `job-source-agent` skips straight to the Chrome Connector; `apply-agent` skips straight to `needs_input`. Otherwise, open the target URL with Playwright MCP.
2. Wait for the page to render fully before reading it — job boards and ATS platforms (Greenhouse, Lever, Workday, etc.) are frequently JavaScript-heavy.
3. **Read the page using Playwright's structured page-read capability** (its accessibility/element snapshot, not a screenshot) to see the actual fields, labels, and content — a screenshot is a last resort, only when the structured read genuinely can't disambiguate something visual. Extract:
   - **Sourcing**: just title, company, posting URL, and application URL — enough to mint a ledger row and let the user recognize/pick it later. Nothing more is persisted at this stage; there's no details file yet for `job-source-agent` to write into.
   - **Applying**: every form field with its label, type, and required/optional status; exact text of any custom question; upload field locations and accepted types; consent/attestation checkboxes; and the description/requirements text itself, verbatim, plus a short company highlight (a few sentences, capped around 40 lines) — this is what `apply-agent` writes into the Posting and Company context sections of the details file once the job is done.
4. If the page fails to load, stop, don't retry blindly, and record it per the section above — then `job-source-agent` switches to the Chrome Connector, `apply-agent` reports `needs_input`. If instead it shows a login wall, ask the user to log in right there in the Playwright session and continue once confirmed — only reach for the Chrome Connector path (switch or `needs_input`, per agent) if that genuinely isn't workable (see "Browsing tool order" above).
5. Close the tab once a job is done with, whether it succeeded, failed, or was flagged.

## Token and time discipline

A multi-step ATS form can rack up more cost than the rest of the apply flow combined if every step re-reads the whole page from scratch. Prefer Playwright's structured read over a screenshot every time it's sufficient — most form-filling never needs vision at all. Cap retries on a stubborn field at two attempts before flagging it rather than trying a third or fourth approach. A lost connection is an immediate stop, never a retry loop — checkpoint the job `failed` with a note and let the next batch pick it up fresh.

## Hard per-job action ceiling

A normal job, done efficiently, finishes in roughly 10-15 browser actions. If a single job passes about 20 without reaching submission, stop and flag it (`needs_input`, "exceeded per-job action budget") rather than continuing to spend against it — this is an operational ceiling on tool calls, not a literal measurement of account usage, but it keeps the worst case for one job bounded rather than open-ended.

## Browser-scoped tools only

No agent in this plugin ever uses full computer-use/desktop-control tools for browsing itself — Playwright MCP and the Chrome Connector are both browser-scoped by nature. The one narrow exception is native file-attach dialogs when using the Chrome Connector (relevant to `job-source-agent`'s toolset, and to `apply-agent` only in the rare case the Chrome Connector has been explicitly approved for one job) — see `skills/file-upload/SKILL.md`. This browsing path is for job-application sites only — **Notion is never accessed this way**; see `skills/notion-tracker/SKILL.md`.

## Anti-patterns to avoid
- Rewriting or summarizing job description text during extraction — copy it verbatim.
- Guessing at a field's purpose when it's ambiguous — flag it instead.
- Reaching for the Chrome Connector just because a page is slow, or switching to it silently.
- Retrying Playwright on a domain already recorded as unable to load.
- Hardcoding a list of sites that "need" the Chrome Connector — always decide from what the page actually shows, not a name match.
- Taking a screenshot when Playwright's structured page read already answers the question.
- Filling a real question with anything not grounded in `identity/` or the posting, or submitting while something needed is still missing.
- Clicking a third-party quick-apply button ("Apply with LinkedIn," "Apply with Indeed," or similar) when the page also offers a plain form — the plain form is always the right path when both exist; see `agents/apply-agent.md`.
- Using this browser tooling to reach Notion for any reason — that's `tracker-sync-agent`'s job, via the connector MCP, never a browser tab.
- `apply-agent` trying to work around not having the Chrome Connector or Gmail connector instead of reporting `needs_input` — it doesn't have either by default, and that's by design, not an oversight to route around.
