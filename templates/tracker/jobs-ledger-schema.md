# Jobs Ledger Schema

Every job lives as one row in a single flat ledger file, plus — once `apply-agent` has actually finished that job — one details file. Every NemoHire command and agent that touches job data reads and writes these two things, nothing else.

## Why this exists

Scattering one job across up to five separate small files (posting, company context, resume, cover letter, application record), plus a nested JSON structure for run/batch state, meant that touching a single job's status required reading and rewriting an entire JSON tree, and finding a job's data meant knowing which of several folders to look in. Both of those turned into real failures in practice: a single status update burning a full-file rewrite, and `apply-agent` losing track of its own file layout mid-job and resorting to scanning the filesystem with `find` to locate a file that should have been at one deterministic path. This schema fixes both: one append-only, line-per-job ledger that's cheap to filter and cheap to update one row at a time, and one predictable details file per job.

## Path resolution — read this before touching any file

Every path below is relative to **the project root — the directory that contains `.claude/`**. That is always the working directory for `Read`/`Write`/`Edit`/`Bash` calls in this plugin. Never assume `.claude/nemohire/` itself is the working directory, and never assume a bare path like `jobs/jobs.jsonl` resolves correctly on its own — always use the full path starting with `./.claude/nemohire/`. When dispatching another agent (a `Task` call), reference a specific file with `@.claude/nemohire/jobs/jobs.jsonl` or `@.claude/nemohire/jobs/details/<seq>-<id>.md` in the dispatch instruction — the `@` form is resolved by Claude Code against the project root before the dispatched agent starts reasoning about paths at all, which is the most reliable way to hand off a specific file.

If a file isn't where it's supposed to be at its one deterministic path, that is a real error — report the job `failed` with that detail. **Never fall back to `find` or any other filesystem-wide search.** Every path in this plugin is deterministic from a job's `id`/`seq`; there is nothing to search for. The one exception: a job's details file legitimately doesn't exist yet while its `st` is `new` or `queued` — that's not an error, it's simply not written until `apply-agent` finishes that job, as one of a batch of up to five it may be working through (see below).

## `jobs.jsonl` — the ledger

`./.claude/nemohire/jobs/jobs.jsonl` — one line, one compact JSON object, per job. Never a nested structure; never nice-printed with multi-line formatting. Fields:

```json
{"seq":42,"id":"8f3a2c","st":"queued","co":"Acme Inc","role":"Staff Engineer","url":"https://acme.example/apply/123","ref":"./.claude/nemohire/jobs/details/0042-8f3a2c.md","note":"","at":"2026-07-14T10:32:00Z"}
```

- `seq` — order added, zero-padded to 4 digits everywhere it's used in a filename (`0042`, not `42`). Informational/display only, not a uniqueness guarantee — `id` is the real key. Computed as (current line count of `jobs.jsonl` at the moment of appending) + 1; never worth a dedicated read just to compute it precisely.
- `id` — the short unique token this job is keyed by everywhere (same id names its `ref` file, alongside `seq`).
- `st` — one field covers status and outcome: `new` (sourced, not yet selected to apply), `queued` (selected, pending), `submitted`, `failed`, `needs_input`, or `manual`. Anything other than `new`/`queued` is a terminal outcome — there's no separate "done" flag to track alongside it.
- `co`, `role` — company/role, best-effort, for display only.
- `url` — the URL used to open the application (the application URL if one exists separately from the posting URL, otherwise the posting URL). This is also the dedup key against the tracker.
- `ref` — the deterministic path this job's details file will live at, computed from `seq`+`id` the moment the row is minted. **The file itself doesn't exist yet at that point** — see below. Once `st` reaches a terminal value, `ref` must resolve to a real file; if it doesn't, that's a genuine error.
- `note` — short reason, only meaningful for `failed`/`needs_input`/`manual`; empty string otherwise.
- `at` — ISO-8601, last updated.

### Filtering and pagination — never read the whole file

Use `Grep` against `./.claude/nemohire/jobs/jobs.jsonl` to find rows, with `head_limit` for pagination:
- Pending jobs: pattern `"st":"new"` or `"st":"queued"`.
- A specific job: pattern `"id":"<id>"`.
- Next batch of N queued jobs: pattern `"st":"queued"`, `head_limit: N`.

This returns just the matching lines — never load the whole file into context to filter it yourself.

### Updating a row — one line, never the whole file

Every update (minting a new row, flipping `new`→`queued`, or writing a terminal outcome) is a single `Edit` call: `old_string` is the row's exact current line, `new_string` is the same line with its changed fields. This is a diff-sized edit regardless of how many thousand rows the file holds. Appending a brand-new row works the same way — anchor `old_string` on whatever the current last line is (known from the append that created it, or from a `Grep` for `"seq":<n>` on the previous highest seq) and append the new line after it.

## `jobs/details/<seq>-<id>.md` — one file per job, created only when that job is done

Filename is `<seq>-<id>.md`, zero-padded seq first (e.g. `0042-8f3a2c.md`) — sorts naturally in a plain file listing. **This file does not exist while a job is `new` or `queued`.** `apply-agent` creates it exactly once, in a single write, right at the end of its own dispatch, whatever the outcome — submitted, failed, needs_input, or manual. Nothing pre-seeds it: no Posting-only stub at source time, no placeholder at mint time. This keeps `jobs/details/` an honest reflection of what's actually been worked — a file appears there when, and only when, `apply-agent` has finished with that job.

```markdown
# <id> — <role> @ <company>

## Posting
<what apply-agent found on the actual application page — title, company, description/requirements as encountered, posting URL, application URL. Brief or absent with a note if the page never loaded at all.>

## Company context
<short highlight, capped ~40 lines — written by apply-agent while browsing>

## Cover letter
<written by apply-agent, in the user's voice — omit if the job never reached this step>

## Answers
Q: <question text>
A: <answer given>
(repeat per custom question — omit this section entirely if the form had none, or the job never reached this step)

## Application record
- Files uploaded: <list>
- Submitted: <ISO-8601>
- Outcome: <matches the ledger row's `st`>
- Note: <if any>
```

`apply-agent` reads the actual posting itself by opening `application_url` and browsing it — that's already required to fill the form, so there's no separate posting extract to keep in sync. It composes every section above in memory as it works the job, then writes this file **once**, at the very end, rather than incrementally across several separate writes.

**Exception — tailored resume.** If the user has per-job resume tailoring on (`identity/documents.md`'s "Tailor per job: yes"), the tailored resume is a real document, not markdown prose, and stays its own file: `./.claude/nemohire/jobs/resumes/<seq>-<id>.<ext>`. When tailoring is off (the default), there's no per-job resume file at all — the base resume path from `identity/documents.md` is used directly.

## Who writes what

- **Sourcing** (`job-source-agent`, via `/nemohire:source`): for each posting found, appends one row to `jobs.jsonl` with `st:"new"` — company, role, and both URLs, plus the precomputed `ref` path. One write per posting, nothing else; no details file, since nothing has been applied to yet.
- **`/nemohire:apply`/`/nemohire:continue`** (the top-level commands, directly, not a subagent): read rows with `st:"new"` (or an externally supplied `--jobs-file`, minting fresh rows for entries not already in the ledger), let the user pick, and flip selected rows to `st:"queued"` — a single-line `Edit` per row. This is the only place ids get minted for jobs that didn't come through `/nemohire:source`. They also dispatch `apply-agent` with a batch of jobs at a time (`--batch-size`, default 5, dynamic) — one dispatch per batch, not one per job — and immediately after each batch returns, for every job in it, both edit that job's ledger row to its terminal `st` (and `note` if relevant) **and** append/update its row in `tracker/applications.md` — using the `co`/`role`/`url` they already have from the ledger row plus the outcome/note `apply-agent`'s output contract returns for that job. There's no intermediate agent doing either of these on their behalf. Neither command ever touches `jobs/details/`.
- **`apply-agent`**: given an array of one to five jobs, works through them one at a time, strictly sequentially — never in parallel, never interleaved. For each job it browses `application_url` itself for the posting/form content — nothing to read from `jobs/details/` beforehand, since nothing's written there yet — and writes that job's `jobs/details/<seq>-<id>.md` exactly once, at the end of that job's own pass, before moving to the next job in its batch. Never touches any ledger row or `tracker/applications.md` directly, for any job — all of that is whichever top-level command dispatched it, right after the whole batch returns.

## Resuming

`/nemohire:continue` is the same operation as `/nemohire:apply`, just without picking new jobs first: `Grep` `jobs.jsonl` for `"st":"queued"`, and dispatch `apply-agent` with a batch of jobs at a time, checkpointing every job in a batch once that batch returns. There's no separate run/batch bookkeeping file — a row's `st` value is the entire resumability record. A `failed`/`needs_input`/`manual` row is never silently retried by `/nemohire:continue`; a fresh `/nemohire:apply` against that specific job (once whatever was missing is resolved) is how you retry one.

## Dedup against the tracker

`tracker/applications.md` remains the durable, human/Notion-facing record and the dedup source — `/nemohire:apply` reads it directly (a plain file read) before selecting which `new` rows to queue, dropping any whose URL is already there (normalized). The ledger and the tracker are not the same thing: the ledger is the technical processing queue and pointer index; the tracker is the readable history.
