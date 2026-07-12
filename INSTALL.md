# Installing NemoHire

## Claude Code

1. Add the Nemous Marketplace (or point directly at this plugin directory) and install `nemohire`.
2. Confirm installation:
   ```
   claude plugin list
   ```
3. Run `/nemo:init` inside any project to start building your identity. All NemoHire data is stored under `.claude/nemohire/` in that project, so you can run a job search per-project or in a dedicated "job-search" project.

## Cowork

1. Install the `nemohire.plugin` file from the marketplace (or drag it into a Cowork session).
2. Select a folder for Cowork to work in — this is where `.claude/nemohire/` will live.
3. Run `/nemo:init` (or just ask "let's set up my job search") to begin.
4. For `/nemo:source` and `/nemo:apply`, Cowork will use its built-in browsing first; if a site needs the Chrome Connector, you'll be asked to approve it explicitly, per site.

## Requirements

- For Notion tracking: a connected Notion integration/connector with permission to create databases in your workspace.
- For email sync: a connected mail account (Gmail or similar) with read access.
- For applying: the built-in browser tool, and optionally the Chrome Connector extension for sites that need it.

## First run checklist

- [ ] `/nemo:init` completed — identity files are no longer placeholders
- [ ] `/nemo:init-tracker` completed — tracker backend chosen and created
- [ ] At least one `/nemo:source` run with real target sites/keywords
- [ ] Watched a `/nemo:apply` run closely on one job before trusting it to run unattended on several
- [ ] If a run ever stops partway (missing info, a failure, a closed session), know that `/nemo:continue` picks it back up — no need to restart from scratch

## Uninstalling

Removing the plugin does not delete your data — `.claude/nemohire/` stays in your project as plain markdown/JSON, so your identity and history are portable even if you stop using the plugin.
