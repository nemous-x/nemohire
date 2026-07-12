# NemoHire

An AI-powered hiring copilot for Claude Code and Cowork. NemoHire builds a persistent professional identity for you, then uses it to source jobs and apply to them — live, one question at a time, in your own voice — while tracking everything and keeping your inbox synced with your job search.

## Why identity-first

Generic "write me a cover letter" tools produce generic cover letters. NemoHire's first step (`/nemo:init`) builds a deep, persistent model of who you are — your real experience, achievements, salary rules, writing voice, and reusable interview answers — stored as plain markdown in `.claude/nemohire/identity/`. Everything else reads from this before generating anything, so materials stay grounded in what's actually true about you, and sound like you.

## Commands

The flow is deliberately simple: source jobs, then apply to them. There's no separate ranking or drafting stage — everything happens live, per job, when you apply.

| Command | Purpose | Model |
|---|---|---|
| `/nemo:init` | Build/update your professional identity | Sonnet (`identity-agent`) |
| `/nemo:source` | Find jobs from sites, keywords, roles, locations | Haiku (`job-source-agent`) |
| `/nemo:init-tracker` | Set up Notion or local markdown tracker | Haiku (`tracker-agent`) |
| `/nemo:apply` | Apply sequentially, one job at a time, one question at a time | Haiku (coordination + browsing) + Sonnet (all writing/answering) |
| `/nemo:sync-email` | Classify hiring emails and update the tracker | Haiku (`email-agent`) |

## Architecture

Every capability is split across three layers so it stays maintainable and cheap to extend:

- **Commands** (`commands/*.md`) — user-facing entry points (the table above).
- **Agents** (`agents/*.md`) — 7 single-responsibility subagents: `identity-agent`, `job-source-agent`, `application-coordinator-agent`, `browser-agent`, `tracker-agent`, `memory-agent`, `email-agent`. Only one of them — `identity-agent` — ever writes or answers content a human reads.
- **Skills** (`skills/*/SKILL.md`) — 6 reusable capabilities (browser-navigation, chrome-connector, file-upload, notion-tracker, email-sync, document-generation) that agents invoke rather than re-implementing.
- **Memory** (`.claude/nemohire/` in your project) — markdown-first persistence for identity, sourced/applied jobs, the tracker, and email sync state. See `templates/` for the schema of every file, scaffolded automatically by `/nemo:init` and `/nemo:init-tracker`.

```
.claude/nemohire/
├── identity/        # who you are (14 files) — built by /nemo:init
├── jobs/
│   ├── sourced/      # raw extracted postings
│   ├── applied/       # per job: resume.md, cover-letter.md, application-record.md
│   └── rejected/
├── tracker/          # Notion database link or local markdown table
├── emails/           # last-sync watermark
└── config.md
```

Every job in `jobs/applied/<company>-<role>/` carries a complete `application-record.md` (schema in `templates/tracker/application-record.md`): the exact resume and cover letter content submitted, every question the live form asked and the exact answer given, the company context gathered, files uploaded, and the timestamp/URL. It's a full copy, not a pointer — you can look back at exactly what was said to any company.

## Model routing — strict, by design

Only `identity-agent` runs on Sonnet, and it's the only agent that ever produces a word a human reads. Everything else — `application-coordinator-agent` and `browser-agent` included — runs on Haiku and is limited to sequencing, browsing, mechanical lookups, and relaying. This isn't a soft guideline: the coordinator has no path to answer a question itself, and `browser-agent` never fills a field it wasn't explicitly handed an answer for.

## Browser strategy

NemoHire always tries the built-in browser tooling first (`skills/browser-navigation`). If a site can't be handled that way, it stops and asks your explicit permission before falling back to the Chrome Connector (`skills/chrome-connector`) — it never switches silently. **No agent in this plugin ever uses full computer-use/desktop-control tools — only browser-scoped tools**, for sourcing and for applying alike. Tabs are closed once a job is done with.

## Safety guarantees for `/nemo:apply`

- Applications are processed **strictly sequentially**, never in parallel.
- A **submission summary** (company, role, every answer, every uploaded file, flagged fields) is always shown before the actual submit action fires — even in an unattended batch run.
- No fields are ever filled with fabricated information; anything not present in your identity or the posting is flagged for you rather than guessed.
- Failed submissions leave the tracker status untouched (never marked "Applied" on failure).

### The apply loop: a relay, not a debate

`application-coordinator-agent` (Haiku) never writes or decides an answer — it only sequences and relays. `browser-agent` (Haiku, browser-scoped tools only) is the only agent that touches the page, and it never invents content either. `identity-agent` (Sonnet) is the only agent that ever produces an answer, and it always does so as you:

1. `identity-agent` tailors a resume and writes a cover letter for the job, grounded in the actual posting.
2. `browser-agent` opens the application, pulls the company/product description, fills anything it can answer itself straight from `identity/profile.md`/`identity/documents.md` (name, email, phone, which file to attach), and returns the first field it can't.
3. The coordinator relays that field to `identity-agent`, unchanged and without judgment.
4. `identity-agent` answers as you — first person, plain human grammar, grounded in the company context and the job posting.
5. The coordinator relays that answer back; `browser-agent` fills it in verbatim and returns the next field.
6. Repeat until nothing's left, then uploads, then the submission summary, then submit.

### Human voice, always

Every piece of content that ends up in front of an employer — resume, cover letter, every live answer — is written strictly in the user's own first-person voice, as the candidate, in plain human grammar. It never discloses, hints at, or references AI involvement, and it never breaks character with disclaimers or meta-commentary. Only `identity-agent` produces this content — `application-coordinator-agent` and `browser-agent` structurally cannot, since neither has any path to generate or judge written content.

## Getting started

1. Run `/nemo:init` to build your identity.
2. Run `/nemo:init-tracker` to choose Notion or local markdown.
3. Run `/nemo:source` with your target sites/keywords.
4. Run `/nemo:apply` on the jobs you want to pursue — one at a time, live.
5. Run `/nemo:sync-email` periodically (or on a schedule) to keep the tracker current.

See `INSTALL.md` for setup in Claude Code vs. Cowork.

## Extending NemoHire

- Add a new job board or ATS quirk? Extend `skills/browser-navigation/SKILL.md`, not the commands — commands stay stable.
- Want a different tracker backend (e.g. Airtable)? Add a new skill alongside `notion-tracker`, and extend `tracker-agent`'s backend switch in `tracker/sync-state.json`.
- New identity dimension? Add a template under `templates/identity/`, wire it into `commands/nemo-init.md`'s interview, and reference it from `identity-agent`.
- Tempted to add a new specialized writing agent? Don't — `identity-agent` is deliberately the single content authority. Extend its responsibilities instead of splitting judgment across multiple Sonnet agents again.

Contributions should keep each new command/agent/skill single-responsibility — that's what keeps this plugin maintainable as it grows.
