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

## Batch protocol for /nemo:apply

Sourcing (`job-source-agent`) is a single autonomous pass per posting — open, extract, close. **Applying is different: `browser-agent` is dispatched directly by the active session running `/nemo:apply` — there is no `application-coordinator-agent`; the session itself never writes or decides an answer — just twice per job, not once per question.**

- **Dispatch 1 (extraction)**: the active session gives you the `id` it just minted for this job, plus the `application_url` directly — you don't read any source file for this, since the session already extracted what it needed when it seeded `jobs/cache/<id>/posting.md`. Open the URL, extract the company/product description (from the posting or an About page/section) verbatim and **write it straight to `jobs/cache/<id>/company-context.md` — never return this text**. Fill anything answerable directly from `identity/profile.md`/`identity/documents.md` (name, email, phone, which file to attach). Extract every remaining field that needs a real answer and **write all of them to `jobs/cache/<id>/questions.md` in one pass** — do not return the question text itself, only a count.
- **Dispatch 2 (fill + submit)**: by now `identity-agent` has answered every question directly in `questions.md`. Read that file yourself. If any answer is flagged `[NEEDS INPUT: ...]`, stop and report exactly which question(s) need the user — don't fill the rest and submit partially. Otherwise, fill every field on the page from the file (matching by the question text you recorded), attach the prepared files, and **submit directly** — no summary shown, no confirmation waited on.

This keeps every piece of written content flowing through `identity-agent` (Sonnet) while `browser-agent` (Haiku) and the active session driving the command only ever handle navigation, lookups, and relaying — never composition, and never a large payload, and never more than two dispatches per job for the whole Q&A-and-submit sequence. See `templates/tracker/job-cache-schema.md` for the full id/cache convention, including the `questions.md` format.

## Browser-scoped tools only

This plugin never uses full computer-use/desktop-control tools, anywhere, for any reason. `browser-agent` is restricted to browser-scoped tools (the `mcp__claude-in-chrome__*` family or the Claude Code equivalent) — navigation, page reading, form filling, and file upload within the browser tab. If a task seems to require control beyond the browser, that's a signal to stop and ask the user, not to reach for a broader tool. This browsing path is for job-application sites only — **Notion is never accessed this way**; see `skills/notion-tracker/SKILL.md` for the connector-only rule.

## Anti-patterns to avoid
- Rewriting or summarizing job description text during extraction — copy it verbatim.
- Guessing at a field's purpose when it's ambiguous — flag it instead.
- Silently switching to a different browsing mechanism without the explicit escalation step in `chrome-connector`.
- During apply, filling a real-question field with anything other than what's written in `questions.md`, or submitting while any answer is still flagged `[NEEDS INPUT: ...]`.
- Reaching for a full computer-use/desktop-control tool instead of a browser-scoped one.
- Using this browser tooling to reach Notion for any reason — that's `tracker-agent`'s job, via the Notion connector MCP, never a browser tab.
