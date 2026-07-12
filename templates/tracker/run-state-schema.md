# Run State Schema

This describes `.claude/nemohire/jobs/run-state.json` — the single, central checkpoint ledger for `/nemo:apply` runs. It's what makes `/nemo:continue` possible and what keeps the flow connected across a session: every command that touches an apply run reads and writes this one file rather than inferring progress from scattered folders.

## Why this exists

Before this file existed, the only record of "what happened" was the tracker (updated per job, but with no notion of a queue) and the `jobs/cache/<id>/` / `jobs/applied/<id>/` folders (keyed by id, but with nothing that says which ids belong to the same run, or which job in a run was next). If a run stopped — a `[NEEDS INPUT: ...]` block, an error, the user closing the session — there was no way to resume from where it left off; `/nemo:apply` had to be re-run from scratch, re-picking jobs and (if the user hadn't run `/nemo:init` recently) sometimes re-asking things it should already have known. `run-state.json` fixes both problems: it's the checkpoint for resuming, and it's a central, explicit record of what every command has already done.

## File location and shape

`.claude/nemohire/jobs/run-state.json`:

```json
{
  "current_run": {
    "run_id": "<short unique token, minted once per /nemo:apply invocation>",
    "jobs_file": "<path given via --jobs-file, or the default used>",
    "started_at": "<ISO-8601>",
    "updated_at": "<ISO-8601, bumped on every job checkpoint>",
    "jobs": [
      {
        "id": "<the job id minted for this job — same id used under jobs/cache/<id>/ and jobs/applied/<id>/>",
        "company": "<best-effort, for display only>",
        "role": "<best-effort, for display only>",
        "application_url": "<best-effort, for display only>",
        "status": "queued | done",
        "outcome": "submitted | failed | skipped | needs_input | null",
        "note": "<short reason, only set for failed/skipped/needs_input>",
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

- **The active session running `/nemo:apply`** mints `run_id` at the very start of a run (right after the user picks which jobs to apply to) and writes the initial `jobs` array with every selected job set to `status: "queued"`, `outcome: null`. This happens once, before the per-job loop starts.
- **The active session** updates a single job's entry to `status: "done"` with the appropriate `outcome` (and `note` if relevant) immediately after that job finishes — submitted, failed, skipped, or stopped on `[NEEDS INPUT: ...]` — before moving to the next job. This is the checkpoint: if the run stops for any reason right after this write, `/nemo:continue` knows exactly which jobs are settled and which aren't.
- **`memory-agent`** performs the actual file write/update when delegated to, exactly as instructed — it does not decide statuses or outcomes itself.
- When a run finishes with no `"status": "queued"` jobs left, the next `/nemo:apply` invocation (not `/nemo:continue`) moves `current_run` into `history` and starts a fresh one.

## How `/nemo:continue` uses this

`/nemo:continue` reads `current_run` and finds the first job still marked `"status": "queued"`. It resumes the exact same per-job sequence `/nemo:apply` uses (see `commands/nemo-apply.md`'s "The flow, concretely"), starting from that job — every job already marked `"status": "done"` is skipped outright, regardless of its outcome (a `failed` or `skipped` job is not silently retried; the user is told about it and can re-run `/nemo:apply` fresh against it if they want another attempt). If `current_run` has no queued jobs at all (or doesn't exist), `/nemo:continue` says there's nothing to resume and points the user to `/nemo:apply`.

## One run at a time

If `/nemo:apply` is invoked while `current_run` still has queued jobs (a previous run didn't finish), it tells the user a run is already in progress and offers `/nemo:continue` instead of silently starting a second, overlapping run. The user can still force a fresh run explicitly if they want to abandon the incomplete one.
