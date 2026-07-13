# Installing NemoHire

## Claude Code

1. Add the Nemous Marketplace (or point directly at this plugin directory) and install `nemohire`.
2. Confirm installation:
   ```
   claude plugin list
   ```
3. Run `/nemohire:init` inside any project to start building your identity. All NemoHire data is stored under `./.claude/nemohire/` in that project, so you can run a job search per-project or in a dedicated "job-search" project.

## Cowork

1. Install the `nemohire.plugin` file from the marketplace (or drag it into a Cowork session).
2. Select a folder for Cowork to work in — this is where `./.claude/nemohire/` will live.
3. Run `/nemohire:init` (or just ask "let's set up my job search") to begin.
4. For `/nemohire:source` and `/nemohire:apply`, NemoHire uses **Playwright MCP** by default — make sure it's connected. A login wall is handled right there in Playwright (you're asked to log in in the same browser window), not by switching tools. NemoHire only reaches for the Chrome Connector in two narrower cases: a site the connected Playwright genuinely can't load at all, or a login that genuinely can't be done through Playwright's own session — always with your explicit per-site approval. A site that fails to load once is remembered in `./.claude/nemohire/browser-fallback-sites.json`, so later jobs on that domain go straight to the Chrome Connector instead of retrying and failing again.

## Requirements

- Local tracking works out of the box — no connector required.
- For Notion sync: run `/nemohire:sync-tracker` whenever you want it — it asks for a connected Notion integration the first time, and links or creates a database then. Nothing else in NemoHire waits on or requires this.
- For email sync: a connected mail account (Gmail or similar) with read access.
- For applying: Playwright MCP connected and working. Logins happen right there in Playwright by default. The Chrome Connector extension is used only reactively, for the narrower cases — when a page won't load in Playwright at all, or a login genuinely can't be done through Playwright's own session — and always with your explicit approval per site.
- For applications that email a verification code before they're actually submitted: the Gmail connector, asked about during `/nemohire:init`. Optional — without it, that job is just flagged as needing that input instead of being retrieved automatically.

## First run checklist

- [ ] `/nemohire:init` completed — identity files are no longer placeholders
- [ ] `/nemohire:init-tracker` completed — local tracker created
- [ ] At least one `/nemohire:source` run with real target sites/keywords (optional)
- [ ] Watched a `/nemohire:apply` run closely on one job before trusting it to run unattended on several
- [ ] If a run ever stops partway (missing info, a failure, a closed session), know that `/nemohire:continue` picks it back up — no need to restart from scratch
- [ ] For a large queue (dozens of jobs): `/nemohire:apply`/`/nemohire:continue` process jobs in bounded batches (default 5, `--batch-size <n>`) so token cost doesn't compound across a long run — re-invoke `/nemohire:continue` for more batches whenever convenient, or set it up as a recurring scheduled task (see the `schedule` skill)
- [ ] Know that each job has a hard per-job action ceiling (~20 browser actions) so a single stubborn application can't run away with an outsized share of a session's budget
- [ ] Never start a second `/nemohire:apply`/`/nemohire:continue` (manual or scheduled) against the same project while one is already running — they'd share one browser session
- [ ] Run `/nemohire:sync-tracker` whenever you want your local tracker mirrored into Notion — it's a separate, on-demand step, not automatic

## Uninstalling

Removing the plugin does not delete your data — `./.claude/nemohire/` stays in your project as plain markdown/JSON, so your identity and history are portable even if you stop using the plugin.
