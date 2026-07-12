---
description: Resume an interrupted /nemo:apply run from its last checkpoint — same logic as /nemo:apply, just picking up where it stopped instead of starting a new run.
argument-hint: "[none]"
allowed-tools: Task, Read, Write, Glob
---

# /nemo:continue — Resume an Interrupted Apply Run

This command exists because a `/nemo:apply` run can stop partway through — a `[NEEDS INPUT: ...]` block, a submission failure, an error, the user closing the session — and re-running `/nemo:apply` from scratch would mean re-picking jobs and losing track of what already finished. `/nemo:continue` reads the checkpoint `/nemo:apply` leaves behind and picks up from exactly the next unfinished job. **The per-job logic is identical to `/nemo:apply`'s — this command does not reimplement it.** See `commands/nemo-apply.md`'s "The flow, concretely," steps 4–12; this command runs that same sequence, just sourced from the existing run instead of a fresh jobs file and job selection.

Like `/nemo:apply`, this command's logic runs directly in the active session — there is no coordinator subagent. It dispatches `identity-agent`, `browser-agent`, `tracker-agent`, and `memory-agent` via `Task`, exactly as `/nemo:apply` does.

## Preconditions

1. **Identity must exist and be populated.** Same check as `/nemo:apply` — `.claude/nemohire/identity/` must have real content, not placeholders. If not, stop and point to `/nemo:init`.
2. **A resumable run must exist.** Read `.claude/nemohire/jobs/run-state.json` (schema: `templates/tracker/run-state-schema.md`). If `current_run` doesn't exist, or every job in it is already `"status": "done"`, there's nothing to resume — say so and point to `/nemo:apply` for a fresh run. Don't invent a run or fall back to re-reading a jobs file.

## What it does

1. **Read `run-state.json`'s `current_run`.** Find every job still `"status": "queued"`, in the order they appear. Every job already `"status": "done"` — regardless of `outcome` (`submitted`, `failed`, `skipped`, or `needs_input`) — is left alone. A `failed` or `needs_input` job is **not** silently retried here; report it in the summary so the user knows it's still unresolved, but only re-attempt it if the user explicitly asks (a plain `/nemo:apply` run against that job specifically, once whatever was missing has been resolved, is the way to retry).
2. **Tell the user what's resuming**, briefly: how many jobs are still queued, and (if any) how many earlier jobs in this run are already done and won't be touched.
3. **For each remaining queued job, run the exact same per-job sequence `/nemo:apply` uses** (mint id if not already minted for this entry — reuse the id already in `run-state.json`'s job list rather than minting a new one — prepare materials, extract questions, answer questions, handle missing info, fill/upload/submit, write the application record, update the tracker immediately, checkpoint the job as done). If a job was left mid-sequence (e.g. materials were prepared but questions were never extracted because the session ended right after), just re-run its full sequence from the top — cache files already written are simply overwritten with the same or refreshed content, which is harmless since each stage reads fresh by id anyway.
4. **Checkpoint immediately after each job**, exactly as `/nemo:apply` does — update `run-state.json` before moving to the next queued job, not batched at the end.
5. **Close out the run** once every job is `"status": "done"` — move `current_run` into `history`, same as `/nemo:apply` would.

## Failure handling

Identical to `/nemo:apply`: a `[NEEDS INPUT: ...]` flag stops that job and checkpoints it as `needs_input` rather than guessing; a submission failure checkpoints as `failed` and moves on to the next queued job rather than aborting the whole resume; Notion access (if that's the tracker backend) goes through the connector MCP only, never the browser.

## Output contract

At the end, report per job: which were already done before this resume (skipped over), which were processed just now and their outcome, and whether anything is still unresolved (`needs_input`/`failed`) for the user to act on.

## Model routing

Identical to `/nemo:apply`: the active session handles sequencing, no coordinator subagent; Haiku (`browser-agent`, `tracker-agent`, `memory-agent`) for navigation/tracking/record-keeping; Sonnet (`identity-agent`) for all content and answers.
