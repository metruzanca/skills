# AGENTS.md

Notes for any agent working in this repository.

## Repo layout

- `skills/use_gleam/` — the Gleam skill. `SKILL.md` is the entry point; it
  indexes the topic files (`setup.md`, `fundamentals.md`, `std-lib.md`,
  `state.md`, `architecture.md`, `performance.md`, `testing.md`, `review.md`).
- `skills/use_gpui/` — the GPUI skill. `SKILL.md` is the entry point; it
  indexes the topic files (`setup.md`, `fundamentals.md`, `state.md`,
  `styling.md`, `architecture.md`, `performance.md`, `testing.md`,
  `review.md`).
- `README.md` — what the repo contains and how to install the skills.

## Conventions

- Keep each skill self-contained under its own `skills/<name>/` directory so
  it can be installed independently with `npx skills add ./skills/<name>`.
- Add new skills as new directories under `skills/`, and list them in
  `README.md`.

## Correctness criteria for skill content

Each skill exists to give agents **verified, current** guidance — inaccurate
or outdated API data is worse than no guidance. When editing a skill:

1. **Only include what is verifiable.** Every code snippet must be checked
   against the official sources for that skill's target (see each skill's
   `SKILL.md` for its verification rule). If a function or API cannot be
   confirmed, rewrite it to a confirmed equivalent or delete it — do not
   preserve it "just in case".
2. **Normalize to the current API.** Translate older forms or drop them.
3. **Flag uncertainty.** Version-drifted APIs are either omitted or written
   with an explicit "verify against your pinned version" note. Never invent
   call names.
4. Keep single sources of truth: one topic file owns a concept, and
   cross-links point there instead of duplicating content.