---
description: Install a product changelog (What's New modal, changelog page, admin console) into this repository, adapted to its own stack and design system
argument-hint: '[--path <subdir>] [--no-admin] [--dry-run] [notes...]'
disable-model-invocation: true
---

Install the changelog feature in **this** repository.

Raw arguments: `$ARGUMENTS`

Argument handling:
- `--path <subdir>` — the app lives in a monorepo subdirectory; treat it as the repo root.
- `--no-admin` — skip the admin console (only valid if the user has another publishing
  path; warn that publishing then has no human gate and confirm before proceeding).
- `--dry-run` — do discovery and produce the plan and Adaptation Map, write no feature code.
- Anything else is free-text context from the user (e.g. "clients are external customers",
  "publisher role is `owner`"). Weight it heavily.

## Procedure

Load the `changelog` skill and follow it. Order is not optional:

1. **Discovery** — `references/01-discovery.md`. Survey the repo: data layer, auth,
   tenancy, routing, design system, i18n/RTL, storage, environments, quality gates.
   Read `CLAUDE.md` / `AGENTS.md` / `CONTRIBUTING.md` if present; they outrank the skill's
   stylistic preferences.

2. **Adaptation Map** — fill `assets/adaptation-map.template.md` with real paths and write
   it to `changelog/ADAPTATION-MAP.md`.

3. **Plan and confirm** — present, in under 20 lines: schema location, function location,
   routes to be added, existing components to be reused, the role that will publish, and
   anything you could not determine. Ask the two questions discovery cannot answer:
   *who is the audience* (external customers vs internal staff) and *which role publishes*.
   **Wait for confirmation before writing feature code.** Stop here if `--dry-run`.

4. **Schema** — `references/02-data-model.md`. Entry + receipt tables, all indexes, plus a
   migration if the repo needs one.

5. **Backend** — `references/03-backend-operations.md`. Shared helpers, then user-facing
   operations, then admin operations. Walk the security checklist at the end of that file
   before moving on.

6. **Admin console** — `references/05-admin-console.md`. Before the user UI: it is how you
   create real test data. Skip only if `--no-admin`.

7. **User UI** — `references/04-ui-contracts.md`. Modal mounted in the authenticated
   layout (stepper for up to 3 pending entries, digest beyond that), the shared media
   gallery component used by both modal and page, changelog page, nav entry point,
   unread badge.

8. **Seed** — create one demo draft describing the changelog feature itself. Leave it as a
   draft; publishing is the human's move.

9. **Verify** — `references/07-verification.md`, all phases. Fix what you find, retest,
   and report with `assets/install-report.template.md`.

## Rules

- Do not add runtime dependencies unless the repo genuinely lacks a primitive; say so and
  ask first if it does.
- Do not modify unrelated code, reformat files you did not otherwise touch, or "improve"
  things you noticed along the way. Mention them instead.
- Do not commit unless asked. Never push.
- If discovery finds no user identity system, stop and offer the reduced build
  (public page + per-browser dismissal) rather than inventing auth.
- Report honestly: anything unverified is reported as unverified.
