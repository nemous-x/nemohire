# Job Cache Schema

This describes how `/nemohire:apply` handles job data, now that sourcing and applying are fully decoupled.

## There is no shared schema between sourcing and applying

`/nemohire:source` writes `./.claude/nemohire/jobs/sourced.json` — a plain JSON array, in whatever reasonable shape `job-source-agent` naturally extracts (typically `title`, `company`, `location`, `salary`, `description`, `requirements`, `posting_url`, `application_url` per entry). This is not a contract anything else has to honor.

`/nemohire:apply --jobs-file <path>` accepts **any** file that looks roughly like a list of jobs — `sourced.json`, a hand-written file, an export from somewhere else, field names that don't quite match. It does not require an id, a status field, or any particular shape.

## Ids are minted by the top-level /nemohire:apply command, not sourced from any file

The top-level session running `/nemohire:apply` (or `/nemohire:continue`) mints a fresh id itself, directly, for every selected job **before dispatching any batch** — a mechanical `Write`, not something `apply-batch-agent` or `apply-agent` does. The id never comes from the input file. It's a short, unique token scoped to this specific apply attempt, and it's what everything downstream is keyed by. Minting and cache-seeding happen at the top level specifically so `apply-batch-agent` never has to ingest full posting/company text into its own context just to pass it along — its dispatches to `apply-agent` carry only `{id, application_url}`.

```
./.claude/nemohire/jobs/cache/<id>/
├── posting.md            # written directly by the top-level /nemohire:apply command, right when it mints the id
└── company-context.md    # written by apply-agent while it's browsing — a short highlight, not a copy of the page (see below)

./.claude/nemohire/jobs/applied/<id>/
├── resume.md              # only written if per-job tailoring is on; absent (not a blank/copy) when the base resume is used as-is
├── cover-letter.md
└── application-record.md
```

## How the format flexibility gets absorbed

The moment the top-level command mints an id, it takes whatever fields it could find in that entry and writes a normalized `jobs/cache/<id>/posting.md` (title, company, description, requirements, posting/application URLs, best-effort) — a direct file write, not something requiring the model to reason over the content. **This is the only place arbitrary input format is dealt with.** `apply-agent` works off this one normalized file, by id, regardless of what the original source file looked like.

## One agent, one dispatch, the whole job

There's no longer a hand-off between a browsing agent and a content-writing agent for a single job — `apply-agent` does the whole thing itself, in one dispatch: reads the posting, opens the application, writes whatever cover letter or answers it needs directly in the user's voice (grounded in `identity/`), fills, uploads, submits. `questions.md` as a separate inter-agent file no longer exists; if a real answer is needed, `apply-agent` composes it itself in the moment, the same dispatch that fills the field with it. Anything it can't ground in `identity/` or the posting, it flags as missing rather than guessing — that's what `needs_input` means.

The company-product highlight in `company-context.md` is written for the record (and to inform the cover letter within the same dispatch) — capped at roughly 40 lines, the core of what the company does, not a copy of the page.

## Avoiding duplicate applications across runs

Since ids are minted per apply-attempt and aren't stable across runs, duplicate-application checking is done by **application/posting URL** against the local tracker directly — `/nemohire:apply` reads `tracker/applications.md` itself before building the queue, no agent dispatch needed for that. The tracker's `Job ID` column records which specific `jobs/applied/<id>/` folder a row corresponds to — useful for looking up the full record, but not the dedup key.

## Checkpointing within a single run

This schema is about per-job cache/output files. A separate file, `jobs/run-state.json` (schema: `templates/tracker/run-state-schema.md`), tracks which jobs within one `/nemohire:apply` run are queued vs. done — across every `apply-batch-agent` dispatch that run has used — and is what `/nemohire:continue` reads to resume an interrupted run at the next batch.
