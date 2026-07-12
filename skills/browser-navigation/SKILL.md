---
name: browser-navigation
description: >
  Use whenever NemoHire needs to open a job board or application page, extract job postings,
  identify form fields, or extract application questions using Claude's built-in browser
  tooling. This is the default, first-choice browsing path — try this before ever considering
  the Chrome Connector fallback skill.
---

# Browser Navigation

Use Claude Code's/Cowork's built-in browser tooling as the default path for every job-search browsing task: opening job boards, reading postings, and navigating application forms.

## Steps

1. Open the target URL with the built-in browser tool.
2. Wait for the page to render fully before extracting content — job boards and ATS platforms (Greenhouse, Lever, Workday, etc.) are frequently JavaScript-heavy.
3. Extract exactly what's needed for the calling task:
   - **Sourcing**: title, company, location, salary (if shown), full description text verbatim, requirements, posting URL, application URL.
   - **Applying**: every form field with its label, type, and required/optional status; exact text of any custom question; upload field locations and accepted file types; any consent/attestation checkboxes.
4. If the page fails to load, returns an auth wall, shows a bot-detection/CAPTCHA challenge, or the built-in tool otherwise cannot retrieve real content — **stop**. Do not retry blindly and do not fall back to the Chrome Connector on your own. Report back that this specific site needs Chrome Connector permission, and let the calling command ask the user.
5. Once extraction/navigation for a given job is complete, close the tab. Don't leave a trail of open tabs across a sourcing or apply run.

## Turn-based protocol for /nemo:apply

Sourcing (`job-source-agent`) is a single autonomous pass per posting — open, extract, close. **Applying is different: `browser-agent` is invoked one turn at a time by `application-coordinator-agent` (a pure relay — it never writes or decides an answer), and never advances the form past a question it can't already answer.**

- **First turn for a job**: open the application URL, extract the company/product description (from the posting or an About page/section) verbatim, fill anything answerable directly from `identity/profile.md`/`identity/documents.md` (name, email, phone, which file to attach), identify the first field that needs a real answer, return both the company context and that field, and stop.
- **Every following turn**: you'll be given the exact answer for the field returned last turn — this came from `identity-agent`, relayed by the coordinator. Enter it verbatim, keep filling anything else you can answer yourself, then find and return the next field that needs a real answer, or report that none remain. Never invent, rephrase, or guess at an answer yourself.
- **Upload and submit are their own explicit turns**, only performed when the coordinator's instruction for that turn says so.

This keeps every piece of written content flowing through `identity-agent` (Sonnet) while `browser-agent` and the coordinator (both Haiku) only ever handle navigation, lookups, and relaying — never composition.

## Browser-scoped tools only

This plugin never uses full computer-use/desktop-control tools, anywhere, for any reason. `browser-agent` is restricted to browser-scoped tools (the `mcp__claude-in-chrome__*` family or the Claude Code equivalent) — navigation, page reading, form filling, and file upload within the browser tab. If a task seems to require control beyond the browser, that's a signal to stop and ask the user, not to reach for a broader tool.

## Anti-patterns to avoid
- Rewriting or summarizing job description text during extraction — copy it verbatim.
- Guessing at a field's purpose when it's ambiguous — flag it instead.
- Silently switching to a different browsing mechanism without the explicit escalation step in `chrome-connector`.
- During apply, filling any field that needs a real answer before checking in and receiving one from `identity-agent`.
- Reaching for a full computer-use/desktop-control tool instead of a browser-scoped one.
