# NemoHire

Your AI copilot for job hunting. Tell it who you are once — it finds jobs and applies for you, in your own voice.

## Install

**Claude Code:**
```
/plugin marketplace add https://github.com/nemous-x/nemous.git
/plugin install nemohire@nemous
```
Then run `/nemohire:init` in any project.

**Cowork Desktop:**
1. Open Settings → Plugins → Marketplaces, add `https://github.com/nemous-x/nemous.git`.
2. Install `nemohire` from that marketplace.
3. Pick a folder for it to work in.
4. Run `/nemohire:init` (or just say "let's set up my job search").

Requires Playwright MCP connected for applying to jobs. Notion and email sync are optional.

## Setup

1. `/nemohire:init` — build your profile
2. `/nemohire:init-tracker` — set up your tracker

## Use it

- `/nemohire:source` — find jobs
- `/nemohire:apply` — apply to jobs
- `/nemohire:continue` — resume if interrupted
- `/nemohire:sync-email` — check inbox for updates
- `/nemohire:sync-tracker` — mirror to Notion

## Good to know

Never invents info about you. Always attaches your resume. Pauses for you to log in when needed. Asks permission before touching email or using anything beyond Playwright to browse. Applies to jobs in small batches, not all at once — if a run stops partway, `/nemohire:continue` picks it back up, nothing is repeated. Everything stays local unless you turn on Notion sync. Uninstalling the plugin doesn't delete your data.

More detail: `INSTALL.md` (setup) · `ARCHITECTURE.md` (how it's built)
