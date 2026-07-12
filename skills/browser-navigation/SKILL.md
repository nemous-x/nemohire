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

Sourcing (`job-source-agent`) is a single autonomous pass per posting — open, extract, close. **Applying is different: `browser-agent` is invoked one turn at a time by `application-coordinator-agent`, and never advances the form on its own.**

- **First turn for a job**: open the application URL, extract the company/product description (from the posting or an About page/section) verbatim, identify the first unanswered field/question, return both, and stop. Do not fill anything — no answer has been supplied yet.
- **Every following turn**: you'll be given the exact answer (or file reference) for the field returned last turn. Enter it verbatim, then find and return the next unanswered field/question, or report that none remain. Never invent, rephrase, or guess at an answer yourself — that's `screening-agent`/`cover-letter-agent`'s job, mediated by the coordinator.
- **Upload and submit are their own explicit turns**, only performed when the coordinator's instruction for that turn says so.

This keeps every piece of written content flowing through Sonnet-tier judgment while Haiku only ever handles one small, mechanical step at a time.

## Anti-patterns to avoid
- Rewriting or summarizing job description text during extraction — copy it verbatim.
- Guessing at a field's purpose when it's ambiguous — flag it instead.
- Silently switching to a different browsing mechanism without the explicit escalation step in `chrome-connector`.
- During apply, filling more than one field before checking back in, or filling any field without having been given its answer first.
