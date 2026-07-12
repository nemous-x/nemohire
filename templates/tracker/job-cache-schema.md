# Job Cache Schema

This describes how `/nemo:apply` handles job data, now that sourcing and applying are fully decoupled.

## There is no shared schema between sourcing and applying

`/nemo:source` writes `.claude/nemohire/jobs/sourced.json` — a plain JSON array, in whatever reasonable shape `job-source-agent` naturally extracts (typically `title`, `company`, `location`, `salary`, `description`, `requirements`, `posting_url`, `application_url` per entry). This is not a contract anything else has to honor.

`/nemo:apply --jobs-file <path>` accepts **any** file that looks roughly like a list of jobs — `sourced.json`, a hand-written file, an export from somewhere else, field names that don't quite match. It does not require an id, a status field, or any particular shape. There is deliberately no code path that treats "jobs `/nemo:source` found" differently from "jobs a person handed in" — both go through the exact same reading.

## Ids are minted directly by the active session, not sourced from any file, and not by a coordinator subagent

When the active session running `/nemo:apply` picks a job entry to actually apply to, it mints a fresh id itself **at that moment** — right before it dispatches `browser-agent` for the first time on this job. There is no `application-coordinator-agent` doing this on its behalf; the session driving the command does it directly. The id never comes from the input file (which might not have one, and might not even use the word "id" for anything). It's a short, unique token scoped to this specific apply attempt.

This id is what everything downstream is keyed by:

```
.claude/nemohire/jobs/cache/<id>/
├── posting.md            # written by the active session, right when it mints the id
├── company-context.md    # written by browser-agent, on its first dispatch for this job
└── questions.md          # written by browser-agent (extraction), then filled in by identity-agent (answers)

.claude/nemohire/jobs/applied/<id>/
├── resume.md              # tailored, or the base resume as-is — see the "Tailor per job" preference
├── cover-letter.md
└── application-record.md
```

## How the format flexibility gets absorbed

The moment the active session mints an id for a job, it takes whatever fields it could find in that entry — however the source file happened to name or shape them — and writes a normalized `jobs/cache/<id>/posting.md` (title, company, description, requirements, posting/application URLs, best-effort). **This is the only place arbitrary input format is dealt with.** Everything after that point — `identity-agent` reading the posting, `browser-agent` opening the application, the whole Q&A pass — works off this one normalized file, by id, regardless of what the original source file looked like.

## `questions.md` — how the whole application's Q&A moves in one file, not one Task call per question

Format, per question:

```
## Q1
**Field:** <label>
**Type:** <text|textarea|select|checkbox|...>
**Required:** yes/no
**Question:** <exact wording from the form>
**Answer:** <blank until identity-agent fills it in>

## Q2
...
```

- `browser-agent` writes this file once, on its first dispatch for a job — every field it couldn't answer itself, `Answer` left blank.
- `identity-agent` is dispatched once per job to answer everything in the file, writing directly into each `Answer` field. If it can't answer one, it writes `[NEEDS INPUT: <what's missing>]` instead of guessing.
- `browser-agent` is dispatched a second time to read the answered file and fill the actual form fields, matching by the `Question`/`Field` text it recorded — then it uploads and submits directly, unless it finds a remaining `[NEEDS INPUT: ...]` flag, in which case it stops and reports exactly what's missing.

This is why a 10-question application only costs a handful of Task dispatches total (extract, answer, fill/submit) rather than 10+ round trips — the file is the handoff, not a coordinator agent (there is no such agent; the active session dispatches these directly).

## Company context and questions stay tied to their id

`browser-agent` writes `company-context.md` and `questions.md` once, for whichever id it was just given. `identity-agent` always reads every cache file **fresh, by the id in the current call** — it never assumes context from a previous job in the same run still applies. A run that processes three jobs mints three different ids, and each one's cache directory is independent: change the id, and the posting/context/questions `identity-agent` reads change with it, by construction.

## Avoiding duplicate applications across runs

Since ids are minted per apply-attempt and aren't stable across runs, duplicate-application checking is done by **application/posting URL** against the tracker (`tracker-agent`), not by id. The tracker's `Job ID` column records which specific `jobs/applied/<id>/` folder a row corresponds to — useful for looking up the full record, but not the dedup key.
