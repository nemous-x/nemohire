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

## Anti-patterns to avoid
- Rewriting or summarizing job description text during extraction — copy it verbatim.
- Guessing at a field's purpose when it's ambiguous — flag it instead.
- Silently switching to a different browsing mechanism without the explicit escalation step in `chrome-connector`.
