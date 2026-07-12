# Jobs Schema

This is the schema for `.claude/nemohire/jobs/jobs.json` — the single source of truth for every job NemoHire knows about, whether it was found by `/nemo:source` or handed in directly. It lives in the **user's project**, under `.claude/nemohire/jobs/`, never inside the plugin's own installed files — it's runtime data, not plugin content.

## Why one JSON file instead of markdown

- **You don't have to use `/nemo:source`.** If you already have job postings from somewhere else, hand `/nemo:apply --jobs-file <path>` a JSON file in this same shape (or close to it — `memory-agent` maps best-effort and flags anything it can't map) and it gets merged straight in. Sourcing and applying are decoupled.
- **Every job gets a stable, unique `id`** the moment it enters `jobs.json` — generated once by whichever agent adds it (`job-source-agent` for `/nemo:source`, `memory-agent` for a manual import), and never regenerated. This id is what makes the rest of the architecture cheap: once a job has an id, agents refer to *it* instead of repeating the job's full text on every exchange.

## Shape

```json
{
  "jobs": [
    {
      "id": "acme-staff-engineer-8f3a2c",
      "title": "Staff Engineer",
      "company": "Acme",
      "location": "Remote",
      "salary": "",
      "description": "",
      "requirements": "",
      "posting_url": "",
      "application_url": "",
      "source": "nemohire",
      "status": "sourced",
      "sourced_at": ""
    }
  ]
}
```

- **`id`**: `<company-slug>-<title-slug>-<6-char-hash>`. The hash is derived from `company + title + posting_url`, so re-sourcing or re-importing the same posting always produces the same id — that's what makes deduplication idempotent instead of guesswork.
- **`description`** / **`requirements`**: verbatim text, never rewritten or summarized. This is what `identity-agent` reads (by id) to ground the resume, cover letter, and every live answer.
- **`source`**: `"nemohire"` if `/nemo:source` found it, `"manual"` if it came in through `--jobs-file`.
- **`status`**: `"sourced"` (default, pending apply) → `"applied"` or `"rejected"`. There's no separate ranked/prepared status — those stages don't exist anymore.

## The per-job cache — where live context lives

`.claude/nemohire/jobs/cache/<id>/company-context.md` is written once, by `browser-agent`, the first time it opens a given job's application (turn 0 of `/nemo:apply`) — the company/product description it pulled from the posting or an About page.

This is the key efficiency rule of the whole plugin: **large text is written to a file once and referenced by id afterward — it is never re-sent between agents on every turn.** Concretely:
- `browser-agent` writes the company context straight to this file itself. It does not return the text to the coordinator — just a short confirmation that it's cached.
- `application-coordinator-agent` never holds or forwards the company context or the job posting text. When it relays a question to `identity-agent`, the payload is just `{ id, question }` — nothing else.
- `identity-agent` reads `jobs.json` (for the posting) and `jobs/cache/<id>/company-context.md` (for company context) itself, by id, whenever it needs them — for resume/cover-letter prep and for every live answer.

This keeps every Task dispatch small and bounded, regardless of how many questions a given application asks — the cost of a 10-question form is ten small `{id, question}` round trips, not ten repetitions of the full job description and company profile.

## `jobs/applied/<id>/`

Once a job is submitted, its folder is keyed by `id` (not company-role text, to avoid collisions when two postings share a company and title): `resume.md`, `cover-letter.md`, `application-record.md` — see `templates/tracker/application-record.md`.
