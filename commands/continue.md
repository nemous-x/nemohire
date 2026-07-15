---
description: Resume an interrupted /nemohire:apply run from its last checkpoint, one bounded batch at a time.
argument-hint: "[--batch-size <n>]"
allowed-tools: Task, Read, Write, Edit, Glob, Grep, WebFetch, mcp__playwright__*, mcp__claude-in-chrome__*, mcp__gmail__*
---

# /nemohire:continue — Resume an Interrupted Apply Run

A `/nemohire:apply` run can stop partway — a needs_input, a failure, an error, the session ending with more still queued. This command finds every job still `"st":"queued"` in the ledger and keeps going, using the exact same logic `/nemohire:apply` uses (see `agents/apply-agent.md`, `templates/tracker/jobs-ledger-schema.md`) — it doesn't reimplement anything, and there's no separate run-state file to reconcile: a row's `st` value is the entire resumability record.

## Path resolution

Every path below is relative to **the project root — the directory that contains `.claude/`**. Never assume `.claude/nemohire/` itself is the working directory; always use the full path starting with `./.claude/nemohire/`.

## Preconditions

1. **Identity must exist and be populated** — same check as `/nemohire:apply`.
2. **A resumable job must exist.** `Grep` `./.claude/nemohire/jobs/jobs.jsonl` for `"st":"queued"`. If none, there's nothing to resume — say so and point to `/nemohire:apply`.

## What it does

1. **Find queued rows.** `Grep` `jobs.jsonl` for `"st":"queued"`. Anything already terminal — submitted, failed, needs_input, or manual — is left alone; none of those are silently retried. A plain `/nemohire:apply` run against a specific job, once whatever was missing is resolved, is how you retry one.
2. **Tell the user what's resuming** — how many jobs are queued, how many chunks that is at the current `--batch-size` (default 5).
3. **Process jobs one at a time, directly**, exactly like `/nemohire:apply`: `Grep` for the next `--batch-size` still-`"st":"queued"` rows (noting each one's `co`/`role`/`url` from that result), then for each, in order, **dispatch `apply-agent`** with exactly `{id, seq, application_url}` — reusing the row already minted from the original run, nothing else. Wait for it to fully return, then immediately: `Grep`+`Edit` that job's line in `jobs.jsonl` to its terminal `st`, and append/update its row in `./.claude/nemohire/tracker/applications.md` from the `co`/`role`/`url` you already have plus the outcome/note `apply-agent` returned — same as `/nemohire:apply`'s step 4. Do this before moving to the next job — never two dispatches touching the browser at once. Report a short summary after each chunk.
4. **Close out** once no row is `"st":"queued"` anymore — nothing further to do, every row's terminal `st` is already the record.

If a job comes back `needs_input` naming a missing connector (Chrome Connector or Gmail), it stays queued — resolving it means the user explicitly approving that connector for `apply-agent` (see `commands/apply.md`'s "Browser strategy" section), not something this command retries on its own.

**Never run this alongside another `/nemohire:apply`/`/nemohire:continue` invocation against the same project** — including a scheduled task overlapping a manual run. They'd end up sharing one browser session at the same time.

## Running a large remaining queue without one marathon session

Call `/nemohire:continue` again whenever convenient, or set it up as a recurring scheduled task (see the `schedule` skill). Every row's `st` value makes this safe to stop and restart as many times as needed.

## Model

Every agent in this flow runs on whatever model is selected for the current session.
