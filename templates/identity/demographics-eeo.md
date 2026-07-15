# Demographic / EEO Self-Identification

Some application forms — mostly US-based ATS platforms (Greenhouse, Lever, Workday, SmartRecruiters, etc.) — include a voluntary "Equal Employment Opportunity" section asking about gender, race/ethnicity, veteran status, and disability status. These are almost always optional and never affect eligibility, but a random or guessed answer here is worse than a consistent, correct one. This file is the single source of truth `apply-agent` reads for any of these questions — it never infers, guesses, or leaves this section for later once it's filled in.

- **Gender:** <!-- e.g. Man, Woman, Non-binary, Decline to self-identify -->
- **Sexual orientation:** <!-- only asked by a minority of forms; e.g. Heterosexual/Straight, Gay, Lesbian, Bisexual, Decline to self-identify -->
- **Race / ethnicity:** <!-- use the exact category text the form offers where possible (e.g. "White", "Middle Eastern or North African", "Asian", "Black or African American", "Hispanic or Latino", "Two or more races", "Decline to self-identify"). If a form only offers older US EEO-1 categories with no MENA option, "White" is the closest match for Middle Eastern/North African ancestry. -->
- **Disability status:** <!-- Yes, No, Decline to self-identify -->
- **Veteran status:** <!-- e.g. "I am not a protected veteran", "I am a protected veteran", Decline to self-identify -->

## How apply-agent should use this

- Answer honestly with whatever is filled in above, matching it to whichever option wording the specific form actually offers (forms phrase these differently — e.g. "Male" vs "Man", "I don't have a disability" vs "No" — pick the closest real match, never leave the question unanswered if it's required).
- If a field here is still a placeholder (not filled in) and a form asks that specific question, don't guess — select "Decline to self-identify" / "Prefer not to answer" if the form offers it; if the form makes it a hard-required field with no decline option, flag the job `needs_input` rather than inventing an answer.
- These questions are legally protected categories — never infer any of them from a name, photo, resume content, or anything else. The only source is this file.
