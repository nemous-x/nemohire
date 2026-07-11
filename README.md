# NemoHire

An AI-powered hiring copilot for Claude Code and Cowork. NemoHire builds a persistent professional identity for you, then uses it to source jobs, rank them, prepare tailored application materials, apply on your behalf, track everything, and keep your inbox synced with your job search — a complete job-search operating system.

## Why identity-first

Generic "write me a cover letter" tools produce generic cover letters. NemoHire's first step (`/nemo:init`) builds a deep, persistent model of who you are — your real experience, achievements, salary rules, writing voice, and reusable interview answers — stored as plain markdown in `.claude/nemohire/identity/`. Every other command reads from this before generating anything, so materials stay grounded in what's actually true about you.

## Commands

| Command | Purpose | Model |
|---|---|---|
| `/nemo:init` | Build/update your professional identity | Sonnet |
| `/nemo:source` | Find jobs from sites, keywords, roles, locations | Haiku |
| `/nemo:rank` | Score sourced jobs against your identity | Sonnet |
| `/nemo:init-tracker` | Set up Notion or local markdown tracker | Haiku |
| `/nemo:prepare` | Generate resume, cover letter, screening answers, research per job | Sonnet |
| `/nemo:apply` | Apply sequentially, one job at a time, with a submission summary before each submit | Haiku (nav/upload) + Sonnet (writing) |
| `/nemo:sync-email` | Classify hiring emails and update the tracker | Haiku |

## Architecture

Every capability is split across four layers so it stays maintainable and cheap to extend:

- **Commands** (`commands/*.md`) — user-facing entry points (the table above).
- **Agents** (`agents/*.md`) — 13 single-responsibility subagents (identity, memory, job-source, job-ranking, resume, cover-letter, screening, browser, upload, tracker, email, qa, application-coordinator). Each has one job and a model assignment.
- **Skills** (`skills/*/SKILL.md`) — 6 reusable capabilities (browser-navigation, chrome-connector, file-upload, notion-tracker, email-sync, document-generation) that agents invoke rather than re-implementing.
- **Memory** (`.claude/nemohire/` in your project) — markdown-first persistence for identity, sourced/ranked/prepared/applied jobs, the tracker, and email sync state. See `templates/` for the schema of every file, scaffolded automatically by `/nemo:init` and `/nemo:init-tracker`.

```
.claude/nemohire/
├── identity/        # who you are (14 files) — built by /nemo:init
├── jobs/
│   ├── sourced/      # raw extracted postings
│   ├── ranked/        # scored + justified
│   ├── prepared/       # per-job resume, cover letter, screening answers, research
│   ├── applied/        # submitted
│   └── rejected/
├── tracker/          # Notion database link or local markdown table
├── emails/           # last-sync watermark
└── config.md
```

## Model routing

Haiku handles anything mechanical: browsing, scraping, extraction, classification, file management, tracker/email syncing. Sonnet handles anything that requires judgment: writing, personalization, resume tailoring, cover letters, interview answers, ranking, and QA. This keeps the plugin fast and cheap on the 80% of steps that don't need heavyweight reasoning, while still producing genuinely personalized output where it matters.

## Browser strategy

NemoHire always tries the built-in browser tooling first (`skills/browser-navigation`). If a site can't be handled that way, it stops and asks your explicit permission before falling back to the Chrome Connector (`skills/chrome-connector`) — it never switches silently. Tabs are closed once a job is done with.

## Safety guarantees for `/nemo:apply`

- Applications are processed **strictly sequentially**, never in parallel.
- A **submission summary** (company, role, every answer, every uploaded file, flagged fields) is always shown before the actual submit action fires — even in an unattended batch run.
- No fields are ever filled with fabricated information; anything not present in your identity is flagged for you rather than guessed.
- Failed submissions leave the tracker status untouched (never marked "Applied" on failure).

## Getting started

1. Run `/nemo:init` to build your identity.
2. Run `/nemo:init-tracker` to choose Notion or local markdown.
3. Run `/nemo:source` with your target sites/keywords.
4. Run `/nemo:rank` to see your best matches.
5. Run `/nemo:prepare` on the jobs you want to pursue.
6. Run `/nemo:apply` to submit, one job at a time.
7. Run `/nemo:sync-email` periodically (or on a schedule) to keep the tracker current.

See `INSTALL.md` for setup in Claude Code vs. Cowork.

## Extending NemoHire

- Add a new job board or ATS quirk? Extend `skills/browser-navigation/SKILL.md`, not the commands — commands stay stable.
- Want a different tracker backend (e.g. Airtable)? Add a new skill alongside `notion-tracker`, and extend `tracker-agent`'s backend switch in `tracker/sync-state.json`.
- New identity dimension? Add a template under `templates/identity/`, wire it into `commands/nemo-init.md`'s interview, and reference it from the agent(s) that need it.

Contributions should keep each new command/agent/skill single-responsibility — that's what keeps this plugin maintainable as it grows.
