# NemoHire Config

This file is created by `/nemo:init` under `.claude/nemohire/config.md` in the user's project.

## Tracker backend
- **Backend:** <!-- notion | local, set by /nemo:init-tracker -->

## Browser strategy
- Built-in browser tooling is always attempted first.
- Chrome Connector fallback requires explicit, per-site user permission — never silent.

## Model routing defaults
- Haiku: browsing, extraction, classification, file management, mechanical tracker/email updates.
- Sonnet: writing, personalization, ranking/reasoning, QA judgment calls.

## Paths
- Identity: `.claude/nemohire/identity/`
- Jobs: `.claude/nemohire/jobs/{sourced,ranked,prepared,applied,rejected}/`
- Tracker: `.claude/nemohire/tracker/`
- Emails: `.claude/nemohire/emails/`
