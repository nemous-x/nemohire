# Jobs Ledger Schema

This replaces the old `jobs/sourced.json` + `jobs/run-state.json` + per-job `jobs/cache/<id>/` and `jobs/applied/<id>/` folders with two things: one flat ledger file, and one details file per job. Every NemoHire command and agent that touches job data reads and writes these two things, nothing else.

## Why this exists

Scattering one job across up to five separate small files (posting, company context, resume, cover letter, application record), plus a nested JSON structure for run/batch state, meant that touching a single job's status required reading and rewriting an entire JSON tree, and finding a job's data meant knowing which of several folders to look in. Both of those turned into real failures in practice: a single status update burning a full-file rewrite, and `apply-agent` losing track of its own file layout mid-job and resorting to scanning the filesystem with `find` to locate a file that should have been at one deterministic path. This schema fixes both: one append-only, line-per-job ledger that's cheap to filter and cheap to update one row at a time, and one predictable details file per job.

## Path resolution — read this before touching any file

Every path below is relative to **the project root — the directory that contains `.claude/`**. That is always the working directory for `Read`/`Write`/`Edit`/`Bash` calls in this plugin. Never assume `.claude/nemohire/` itself is the working directory, and never assume a bare path like `jobs/jobs.jsonl` resolves correctly on its own — always use the full path starting with `./.claude/nemohire/`. When dispatching another agent (a `Task` call), reference a specific file with `@.claude/nemohire/jobs/jobs.jsonl` or `@.claude/nemohire/jobs/details/<id>.md` in the dispatch instruction — the `@` form is resolved by Claude Code against the project root before the dispatched agent starts reasoning about paths at all, which is the most reliable way to hand off a specific file.

If a file isn't where it's supposed to be at its one deterministic path, that is a real error — report the job `failed` with that detail. **Never fall back to `find` or any other filesystem-wide search.** Every path in this plugin is deterministic from a job's `id`; there is nothing to search for.

## `jobs.jsonl` — the ledger

`./.claude/nemohire/jobs/jobs.jsonl` — one line, one compact JSON object, per job. Never a nested structure; never nice-printed with multi-line formatting. Fields:

```json
{"seq":42,"id":"8f3a2c","st":"queued","co":"Acme Inc","role":"Staff Engineer","url":"https://acme.example/apply/123","ref":"./.claude/nemohire/jobs/details/8f3a2c.md","note":"","at":"2026-07-14T10:32:00Z"}
```

- `seq` — order added. Informational/display only, not a uniqueness guarantee — `id` is the real key. Computed as (current line count of `jobs.jsonl` at the moment of appending) + 1; never worth a dedicated read just to compute it precisely.
- `id` — the short unique token this job is keyed by everywhere (same id names its `ref` file).
- `st` — one field covers status and outcome: `new` (sourced, not yet selected to apply), `queued` (selected, pending), `submitted`, `failed`, `needs_input`, or `manual`. Anything other than `new`/`queued` is a terminal outcome — there's no separate "done" flag to track alongside it.
- `co`, `role` — company/role, best-effort, for display only.
- `url` — the URL used to open the application (the application URL if one exists separately from the posting URL, otherwise the posting URL). This is also the dedup key against the tracker.
- `ref` — full path to this job's details file.
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

## `jobs/details/<id>.md` — one file per job

`./.claude/nemohire/jobs/details/<id>.md` replaces the old `jobs/cache/<id>/posting.md` + `company-context.md` + `jobs/applied/<id>/cover-letter.md` + `application-record.md` — one file, written in sections:

```markdown
# <id> — <role> @ <company>

## Posting
<verbatim description, requirements, posting URL, application URL — written once, at source/mint time>

## Company context
<short highlight, capped ~40 lines — written by apply-agent while browsing>

## Cover letter
<written by apply-agent, in the user's voice>

## Answers
Q: <question text>
A: <answer given>
(repeat per custom question — omit this section entirely if the form had none)

## Application record
- Files uploaded: <list>
- Submitted: <ISO-8601>
- Outcome: <matches the ledger row's `st`>
- Note: <if any>
```

`apply-agent` composes the Company context / Cover letter / Answers / Application record sections in memory during its one dispatch and writes this file **once**, near the end of the job, rather than incrementally across several separate writes.

**Exception — tailored resume.** If the user has per-job resume tailoring on (`identity/documents.md`'s "Tailor per job: yes"), the tailored resume is a real document, not markdown prose, and stays its own file: `./.claude/nemohire/jobs/resumes/<id>.<ext>`. When tailoring is off (the default), there's no per-job resume file at all — the base resume path from `identity/documents.md` is used directly.

## Who writes what

- **Sourcing** (`job-source-agent`, via `/nemohire:source`): for each posting found, appends one row to `jobs.jsonl` with `st:"new"` and writes one `jobs/details/<id>.md` with just the Posting section filled in. Two writes per posting — never a growing array file rewritten on every append.
- **`/nemohire:apply`** (the top-level command, directly, not a subagent): reads rows with `st:"new"` (or an externally supplied `--jobs-file`, minting fresh rows for entries not already in the ledger), lets the user pick, and flips selected rows to `st:"queued"` — a single-line `Edit` per row. This is the only place ids get minted for jobs that didn't come through `/nemohire:source`.
- **`apply-batch-agent`**: dispatches `apply-agent` with just `{id, application_url}`, and on each return, edits that job's ledger row to its terminal `st` (and `note` if relevant) — one `Edit` call, immediately, per job.
- **`apply-agent`**: reads `jobs/details/<id>.md` for the posting, writes the rest of that same file once near the end of the job, and never touches the ledger row directly — that's `apply-batch-agent`'s job, right after `apply-agent` returns.

## Resuming

`/nemohire:continue` is the same operation as `/nemohire:apply`, just without picking new jobs first: `Grep` `jobs.jsonl` for `"st":"queued"`, batch them, dispatch `apply-batch-agent` per batch. There's no separate run/batch bookkeeping file — a row's `st` value is the entire resumability record. A `failed`/`needs_input`/`manual` row is never silently retried by `/nemohire:continue`; a fresh `/nemohire:apply` against that specific job (once whatever was missing is resolved) is how you retry one.

## Dedup against the tracker

`tracker/applications.md` remains the durable, human/Notion-facing record and the dedup source — `/nemohire:apply` reads it directly (a plain file read) before selecting which `new` rows to queue, dropping any whose URL is already there (normalized). The ledger and the tracker are not the same thing: the ledger is the technical processing queue and pointer index; the tracker is the readable history.
