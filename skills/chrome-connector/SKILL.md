---
name: chrome-connector
description: >
  Browser access via the user's own real, already-open Chrome — a reactive fallback only, reserved
  for a page that genuinely can't load in Playwright MCP at all, or a login that genuinely can't
  be done inside Playwright's own browser session. It is not the default response to a login wall
  — most logins happen directly in Playwright, in the same window that's already open. Always
  requires the user's explicit go-ahead for that specific site. Never invoke this skill
  proactively, and never based on a hardcoded list of site names.
---

# Chrome Connector

## Not every agent even has this tool

`job-source-agent` carries the Chrome Connector in its own toolset, so for sourcing it genuinely switches to it in the two cases below. `apply-agent` does not — by design, so that the vast majority of jobs (which never need it) don't pay for its tool schema on every dispatch. When `apply-agent` hits one of these two cases, it can't switch tools itself; it stops the job as `needs_input` with a note naming the Chrome Connector explicitly, so the user can see it and approve enabling it (a deliberate, explicit change to `apply-agent`'s own tool list, not something granted silently or by default).

## When this gets used

Only reactively, never proactively, never by name-matching a domain against a fixed list, and never as the automatic response to every login wall — logging in normally happens right inside Playwright's own browser window instead (see `skills/browser-navigation/SKILL.md`). This skill is reserved for two narrower cases:

1. **A page genuinely won't load in Playwright MCP at all.** Timeout, block, broken rendering — `browser-navigation` was tried first and genuinely failed. Record the domain in `./.claude/nemohire/browser-fallback-sites.json` (see `skills/browser-navigation/SKILL.md`) so future jobs on the same domain come straight here instead of failing Playwright again first.
2. **A login genuinely can't be done through Playwright's session.** Most login walls are handled by simply asking the user to log in directly in the Playwright browser that's already open — that's the default, not this. Reach for the Chrome Connector instead only when that's not workable: the session isn't one the user can interact with directly, or the site specifically requires the user's own already-authenticated browser profile (saved passkeys, a trusted-device history, etc.) rather than accepting a fresh login. When this is the case, ask the user to log into that site in the Chrome Connector's browser (their real, already-open Chrome, carrying their actual session), then continue once they confirm.

Either way: always per-site, always with the user's explicit yes for that site, never chained automatically, never treated as blanket permission for every site afterward just because it was granted once.

## Steps
1. Confirm the Chrome extension/connector is actually connected. If not, tell the user to install/connect it and stop.
2. If this is the login case, ask the user to log into the relevant site in that browser first, and wait for confirmation before continuing.
3. Navigate and read using the Chrome Connector's tools — its structured page read first, a screenshot only when that's genuinely not enough. It's one browser, one set of tabs — never driven by more than one dispatch at once, same as the primary path.
4. **File uploads work differently here than in Playwright** — see `skills/file-upload/SKILL.md`. The Chrome Connector doesn't have Playwright's native file-attach capability, so this is the one place a native OS file dialog is expected and handled directly via desktop control, not avoided.
5. Treat any link encountered inside the page (ads, unrelated navigation) with the same suspicion rules as the rest of the plugin.
6. Close tabs when done with a given job.

## Reporting
Say so explicitly in the outcome for that job (e.g. "used the Chrome Connector — page wouldn't load in Playwright" or "used the Chrome Connector — site required login") so the user always knows when it was engaged and why.
