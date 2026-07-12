# NemoHire

An AI-powered hiring copilot for Claude Code and Cowork. NemoHire builds a persistent professional identity for you, then uses it to source jobs and apply to them — in your own voice, submitting automatically once every question has a real answer — while tracking everything and keeping your inbox synced with your job search.

## Why identity-first

Generic "write me a cover letter" tools produce generic cover letters. NemoHire's first step (`/nemo:init`) builds a deep, persistent model of who you are — your real experience, achievements, salary rules, writing voice, and reusable interview answers — stored as plain markdown in `.claude/nemohire/identity/`. Everything else reads from this before generating anything, so materials stay grounded in what's actually true about you, and sound like you.

## Commands

The flow is deliberately simple: source jobs, then apply to them. There's no separate ranking or drafting stage — everything happens live, per job, when you apply.

| Command | Purpose | Model |
|---|---|---|
| `/nemo:init` | Build/update your professional identity | Sonnet (`identity-agent`) |
| `/nemo:source` | Find jobs from sites, keywords, roles, locations | Haiku (`job-source-agent`) |
| `/nemo:init-tracker` | Set up Notion or local markdown tracker | Haiku (`tracker-agent`) |
| `/nemo:apply` | Apply sequentially, one job at a time, question batch answered and submitted in one pass — coordinated by the active session itself, no coordinator subagent | Active session (sequencing) + Haiku (browsing/tracking) + Sonnet (all writing/answering) |
| `/nemo:sync-email` | Classify hiring emails and update the tracker | Haiku (`email-agent`) |

## Architecture

Every capability is split across three layers so it stays maintainable and cheap to extend:

- **Commands** (`commands/*.md`) — user-facing entry points (the table above). `/nemo:apply`'s own sequencing logic runs directly in the active session — there's no dedicated coordinator agent it delegates that to.
- **Agents** (`agents/*.md`) — 6 single-responsibility subagents: `identity-agent`, `job-source-agent`, `browser-agent`, `tracker-agent`, `memory-agent`, `email-agent`. Only one of them — `identity-agent` — ever writes or answers content a human reads.
- **Skills** (`skills/*/SKILL.md`) — 6 reusable capabilities (browser-navigation, chrome-connector, file-upload, notion-tracker, email-sync, document-generation) that agents invoke rather than re-implementing.
- **Memory** (`.claude/nemohire/` in your project) — plain files, scaffolded automatically by `/nemo:init` and `/nemo:init-tracker`. See `templates/` for the schema of every file.

```
.claude/nemohire/
├── identity/                # who you are (14 files) — built by /nemo:init
├── jobs/
│   ├── sourced.json           # optional: whatever /nemo:source found — a plain array, no fixed schema
│   ├── cache/<id>/             # posting.md + company-context.md + questions.md, per apply-attempt id
│   └── applied/<id>/           # resume.md, cover-letter.md, application-record.md
├── tracker/                 # Notion database link or local markdown table
├── emails/                  # last-sync watermark
└── config.md
```

### Sourcing and applying don't share a schema

`/nemo:source` writes `jobs/sourced.json` — a plain array, whatever shape `job-source-agent` naturally extracts. `/nemo:apply --jobs-file <path>` accepts that file, a hand-written one, or an export from somewhere else, all identically — there's no special-casing between "found by NemoHire" and "handed in by you." You don't need to run `/nemo:source` at all.

### Ids are minted per apply-attempt, not read from any file, and not by a coordinator subagent

The moment `/nemo:apply`'s active session is about to actually apply to a specific job, it mints a fresh id right then — itself, in-session, with no separate coordinator agent involved — and writes `jobs/cache/<id>/posting.md` from whatever fields it found in the input entry — title, company, description, requirements, URLs, best-effort. That's the one place arbitrary input format gets absorbed. Everything downstream — `identity-agent` reading the posting, `browser-agent` opening the application, the whole live Q&A loop — works off that one normalized file, by id, regardless of what the original file looked like or whether it had an id at all.

That id is also the backbone of the apply loop's efficiency: large text (the posting, the company/product context, the full set of application questions) is written once, to files keyed by that id, and read directly by whichever agent needs it. It's never re-transmitted between agents question by question — a 10-question application costs a handful of dispatches (extract, answer, fill/submit), not ten repetitions of the full posting and company profile. And because each job gets its own fresh id, `identity-agent` always reads the right context: change the id, and the posting/company-context/questions files it reads change with it — there's no way for one job's context to leak into another's.

Every job in `jobs/applied/<id>/` carries a complete `application-record.md`: the exact resume and cover letter content submitted, every question the form asked and the exact answer given, the company context gathered, files uploaded, and the timestamp/URL. It's a full copy, not a pointer — you can look back at exactly what was said to any company. Since ids aren't stable across runs, duplicate-application checking is done by application/posting URL against the tracker, not by id.

### `/nemo:apply` never re-applies to a job you've already applied to

Before it shows you anything, `/nemo:apply` asks `tracker-agent` for every application/posting URL already in the tracker (whichever backend is active — see below) and drops any matching entry from the jobs file. This runs once per invocation, up front, so you can safely re-run `/nemo:apply --all` against the same sourced file repeatedly without worrying about duplicate applications. The tracker is also updated immediately after each individual job is submitted, not batched until the end of the run — so this filter stays accurate even if a run is interrupted partway through.

### The tracker backend is config, not just internal state

`/nemo:init-tracker` records which backend is active (`notion` or `local`) in `.claude/nemohire/config.md`, alongside the machine-readable copy in `tracker/sync-state.json`. It's information about how your NemoHire setup is configured, not just a private implementation detail — so it lives in config.md where the rest of your setup is documented.

### Notion is connector-only, never browser

Whenever the active backend is `notion`, `tracker-agent` reads and writes it exclusively through the Notion connector MCP tools — creating the database, querying rows, inserting or updating them. Neither `tracker-agent`, `browser-agent`, nor the active session ever opens notion.so in a browser tab or reaches Notion through any browser-scoped or computer-use tool; there's no fallback path through the browser for this backend at all. If the connector isn't authorized, `tracker-agent` stops and says so rather than trying anything else.

## Model routing — strict, by design

`/nemo:apply`'s sequencing — reading the jobs file, minting ids, seeding cache files, dispatching the other agents in order — runs directly in the active session, not in a dedicated coordinator subagent. Only `identity-agent` runs on Sonnet, and it's the only agent that ever produces a word a human reads. Everything dispatched via `Task` — `browser-agent`, `tracker-agent`, `memory-agent`, `job-source-agent`, `email-agent` — runs on Haiku and is limited to browsing, mechanical lookups, and relaying. This isn't a soft guideline: the active session has no path to answer a question itself, and `browser-agent` never fills a field it wasn't explicitly handed an answer for.

## Browser strategy

NemoHire always tries the built-in browser tooling first (`skills/browser-navigation`). If a site can't be handled that way, it stops and asks your explicit permission before falling back to the Chrome Connector (`skills/chrome-connector`) — it never switches silently. **No agent in this plugin ever uses full computer-use/desktop-control tools — only browser-scoped tools**, for sourcing and for applying alike. Tabs are closed once a job is done with. This browser path is for job-application sites only — Notion never goes through it (see above).

## Safety guarantees for `/nemo:apply`

- Applications are processed **strictly sequentially**, never in parallel.
- No fields are ever filled with fabricated information; anything not present in your identity or the posting is written as `[NEEDS INPUT: ...]` and stops the run for that job rather than being guessed.
- There is **no pre-submit summary or confirmation gate** — `browser-agent` submits directly as soon as every question has a real answer. The full record (posting, company context, every question and answer, files uploaded) is preserved in `jobs/cache/<id>/` and `jobs/applied/<id>/application-record.md` for after-the-fact review instead.
- Failed submissions leave the tracker status untouched (never marked "Applied" on failure).

### The apply loop: two file-mediated dispatches per job, run by the active session — not a relay per question, and not a separate coordinator agent

The active session running `/nemo:apply` never writes or decides an answer, and never holds large text for more than a moment — it mints ids, seeds cache files, and dispatches the other agents via `Task`, directly, in the same session the user is talking to. There is no `application-coordinator-agent` it hands this off to. `browser-agent` (Haiku, browser-scoped tools only) is the only agent that touches the page, and it never invents content either. `identity-agent` (Sonnet) is the only agent that ever produces an answer, and it always does so as you:

0. Before any of this, the active session dispatches `tracker-agent` to ask which application/posting URLs are already tracked, and drops those jobs from consideration entirely.
1. The active session mints a fresh `id` for this job itself and writes `jobs/cache/<id>/posting.md` from whatever it found in the input file.
2. `identity-agent` is given just the `id`. It reads `jobs/cache/<id>/posting.md`, always writes a cover letter, and only tailors a resume if `identity/documents.md` says the user opted into per-job tailoring during `/nemo:init` (default: no — the base resume is used as-is). Both saved to `jobs/applied/<id>/`.
3. `browser-agent` is given `{id, application_url}` — **dispatch 1**. It opens the URL, writes the company/product description straight to `jobs/cache/<id>/company-context.md`, fills anything it can answer itself from `identity/profile.md`/`identity/documents.md`, and writes every remaining question to `jobs/cache/<id>/questions.md` in one pass, reporting back only a count.
4. `identity-agent` is given just the `id` again. It reads `posting.md`, `company-context.md`, and `questions.md`, and answers every question in one dispatch, writing directly into each `Answer` field — first person, plain human grammar. Anything it can't ground gets `[NEEDS INPUT: <what's missing>]` instead of a guess.
5. If any answer is flagged `[NEEDS INPUT: ...]`, the active session stops and asks the user for exactly that — nothing downstream is allowed to guess.
6. `browser-agent` is dispatched a second time — **dispatch 2**. It reads the now-answered `questions.md`, fills every field on the page by matching the recorded question text, uploads the prepared files, and **submits directly** — no summary, no confirmation wait.
7. `memory-agent` writes `jobs/applied/<id>/application-record.md`, the tracker is updated immediately (via `tracker-agent`, through the Notion connector MCP if that's the active backend), and `browser-agent` closes the tab(s).

No step in this loop ever repeats the job posting, company description, or question text through a Task call more than once — everything is written to disk once and read by id as many times as needed. And no step in it is delegated to a coordinator subagent — the sequencing happens in the same session the user is running the command from.

### Human voice, always

Every piece of content that ends up in front of an employer — resume, cover letter, every live answer — is written strictly in the user's own first-person voice, as the candidate, in plain human grammar. It never discloses, hints at, or references AI involvement, and it never breaks character with disclaimers or meta-commentary. Only `identity-agent` produces this content — neither the active session's own sequencing logic nor `browser-agent` can produce it, since neither has any path to generate or judge written content.

## Getting started

1. Run `/nemo:init` to build your identity.
2. Run `/nemo:init-tracker` to choose Notion or local markdown.
3. Run `/nemo:source` with your target sites/keywords.
4. Run `/nemo:apply` on the jobs you want to pursue — one at a time, each submitted automatically once fully answered.
5. Run `/nemo:sync-email` periodically (or on a schedule) to keep the tracker current.

See `INSTALL.md` for setup in Claude Code vs. Cowork.

## Extending NemoHire

- Add a new job board or ATS quirk? Extend `skills/browser-navigation/SKILL.md`, not the commands — commands stay stable.
- Want a different tracker backend (e.g. Airtable)? Add a new skill alongside `notion-tracker`, and extend `tracker-agent`'s backend switch in `tracker/sync-state.json`.
- New identity dimension? Add a template under `templates/identity/`, wire it into `commands/nemo-init.md`'s interview, and reference it from `identity-agent`.
- Tempted to add a new specialized writing agent? Don't — `identity-agent` is deliberately the single content authority. Extend its responsibilities instead of splitting judgment across multiple Sonnet agents again.

Contributions should keep each new command/agent/skill single-responsibility — that's what keeps this plugin maintainable as it grows.
