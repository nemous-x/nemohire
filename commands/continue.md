---
description: Resume an interrupted /nemohire:apply run from its last checkpoint, one bounded batch at a time.
argument-hint: "[--batch-size <n>]"
allowed-tools: Task, Read, Write, Glob, WebFetch, mcp__playwright__*, mcp__claude-in-chrome__*, mcp__gmail__*
---

# /nemohire:continue — Resume an Interrupted Apply Run

A `/nemohire:apply` run can stop partway — a needs_input, a failure, an error, the end of one batch with more still queued. This command reads the checkpoint it leaves behind and picks up from the next unfinished job, one batch at a time, using the exact same logic `/nemohire:apply` uses (see `agents/apply-batch-agent.md`, `agents/apply-agent.md`) — it doesn't reimplement anything.

## Preconditions

Actually check with `Read`/`Glob` against `./.claude/nemohire/` — relative to the current project's working directory, **not** `~/.claude` — now, never assume from earlier in the conversation.

1. **Identity must exist and be populated** — same check as `/nemohire:apply`.
2. **A resumable run must exist.** Read `jobs/run-state.json`. If `current_run` doesn't exist, or every job in it is already `"status": "done"`, there's nothing to resume — say so and point to `/nemohire:apply`.

## What it does

1. **Read `current_run`.** Find every job still `"status": "queued"`. Anything already `"status": "done"` — submitted, failed, needs_input, or manual — is left alone; none of those are silently retried. A plain `/nemohire:apply` run against a specific job, once whatever was missing is resolved, is how you retry one.
2. **Tell the user what's resuming** — how many jobs are queued, how many batches that is at the current `--batch-size` (default 5).
3. **Process in batches, one at a time**, exactly like `/nemohire:apply`: take the next `--batch-size` still-queued entries and dispatch `apply-batch-agent` with just their `{id, application_url}` pairs — reusing ids already minted and seeded in `jobs/cache/<id>/` from the original run, nothing else. Wait for it to fully return before dispatching the next — never two batches in flight together.
4. **Close out** once every job is `"status": "done"` — move `current_run` into `history`.

**Never run this alongside another `/nemohire:apply`/`/nemohire:continue` invocation against the same project** — including a scheduled task overlapping a manual run. They'd end up sharing one browser session at the same time.

## Running a large remaining queue without one marathon session

Call `/nemohire:continue` again whenever convenient, or set it up as a recurring scheduled task (see the `schedule` skill). `run-state.json` makes this safe to stop and restart as many times as needed.

## Model

Every agent in this flow runs on whatever model is selected for the current session.
