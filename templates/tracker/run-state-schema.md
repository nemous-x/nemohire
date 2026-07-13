# Run State Schema

This describes `./.claude/nemohire/jobs/run-state.json` — the single, central checkpoint ledger for `/nemohire:apply` runs. It's what makes `/nemohire:continue` possible and what keeps the flow connected across a session: every command that touches an apply run reads and writes this one file rather than inferring progress from scattered folders.

## Why this exists

Before this file existed, the only record of "what happened" was the tracker (updated per job, but with no notion of a queue) and the `jobs/cache/<id>/` / `jobs/applied/<id>/` folders (keyed by id, but with nothing that says which ids belong to the same run, or which job in a run was next). If a run stopped — a `[NEEDS INPUT: ...]` block, an error, the user closing the session — there was no way to resume from where it left off; `/nemohire:apply` had to be re-run from scratch, re-picking jobs and (if the user hadn't run `/nemohire:init` recently) sometimes re-asking things it should already have known. `run-state.json` fixes both problems: it's the checkpoint for resuming, and it's a central, explicit record of what every command has already done.

## File location and shape

`./.claude/nemohire/jobs/run-state.json`:

```json
{
  "current_run": {
    "run_id": "<short unique token, minted once per /nemohire:apply invocation>",
    "jobs_file": "<path given via --jobs-file, or the default used>",
    "batch_size": "<--batch-size given, or the default (5) used for this run>",
    "started_at": "<ISO-8601>",
    "updated_at": "<ISO-8601, bumped on every job checkpoint>",
    "jobs": [
      {
        "id": "<the job id minted for this job — same id used under jobs/cache/<id>/ and jobs/applied/<id>/>",
        "company": "<best-effort, for display only>",
        "role": "<best-effort, for display only>",
        "application_url": "<best-effort, for display only>",
        "status": "queued | done",
        "outcome": "submitted | failed | skipped | needs_input | manual | null",
        "note": "<short reason, only set for failed/skipped/needs_input/manual>",
        "updated_at": "<ISO-8601>"
      }
    ]
  },
  "history": [
    "<previous current_run objects, appended here once a run finishes with no queued jobs left>"
  ]
}
```

## Who writes what

- **The active session running `/nemohire:apply`** mints `run_id` at the very start of a run (right after the user picks which jobs to apply to), mints each job's own `id` at the same time, seeds `jobs/cache/<id>/posting.md` for each one directly, and writes the initial `jobs` array with every selected job set to `status: "queued"`, `outcome: null`. This all happens once, up front, before any batch is dispatched — none of it is `apply-batch-agent`'s job, which keeps its own dispatch context minimal (see `agents/apply-batch-agent.md`).
- **`apply-batch-agent`** writes each job's entry itself, directly, right after `apply-agent` returns an outcome for it — `status: "done"` with the appropriate `outcome` (and `note` if relevant): submitted, failed, needs_input (including a login wall the user wasn't available to clear this run — never silently retried, but never abandoned either, since `/nemohire:continue` picks it back up), or manual (an application that needs a brand-new account **created**, or a multi-step process a person should finish by hand — spotted before any real automation attempt; a login to an existing account is never `manual`) — before moving to the next job in the batch. This is the checkpoint: if the batch (or the whole session) stops for any reason right after this write, the next batch dispatch or `/nemohire:continue` knows exactly which jobs are settled and which aren't.
- When a run finishes with no `"status": "queued"` jobs left, the next `/nemohire:apply` invocation (not `/nemohire:continue`) moves `current_run` into `history` and starts a fresh one.

## How `/nemohire:continue` uses this

`/nemohire:continue` reads `current_run`, groups every job still marked `"status": "queued"` into batches of `batch_size`, and dispatches `apply-batch-agent` for each batch in turn — the same per-job sequence `/nemohire:apply`'s batches use (see `agents/apply-batch-agent.md`), just sourced from the existing run. Every job already marked `"status": "done"` is skipped outright, regardless of its outcome (a `failed` or `skipped` job is not silently retried; the user is told about it and can re-run `/nemohire:apply` fresh against it if they want another attempt). If `current_run` has no queued jobs at all (or doesn't exist), `/nemohire:continue` says there's nothing to resume and points the user to `/nemohire:apply`. A single `/nemohire:continue` invocation doesn't have to process every remaining batch either — it can be re-invoked as many times as needed, including from a scheduled task.

## One run at a time

If `/nemohire:apply` is invoked while `current_run` still has queued jobs (a previous run didn't finish), it tells the user a run is already in progress and offers `/nemohire:continue` instead of silently starting a second, overlapping run. The user can still force a fresh run explicitly if they want to abandon the incomplete one.
