---
description: Verify this repo's changelog install — derives a test matrix from the actual implementation, runs it, fixes what it finds, and reports honestly
argument-hint: '[--backend-only] [--ui-only] [--no-fix]'
disable-model-invocation: true
---

Verify the changelog implementation in **this** repository.

Raw arguments: `$ARGUMENTS`

Argument handling:
- `--backend-only` — Phases 0–2 only.
- `--ui-only` — Phases 0, 1, 3, 4.
- `--no-fix` — report findings without changing code.

## Procedure

Load the `changelog` skill and follow `references/07-verification.md` end to end.

1. **Phase 0 — derive the matrix.** Read `changelog/ADAPTATION-MAP.md` and resolve the
   concrete commands for typecheck, lint, test, running the app, invoking a backend
   function from the terminal, and obtaining two distinct test users plus one admin.
   If you cannot obtain two users, solve that first — per-user isolation is the central
   guarantee and it is untestable with one account.

2. **Phase 1 — static.** Build gates, schema and indexes present, then read every
   changelog file hunting specifically for the four classic invariant violations: a
   user-facing operation accepting a `userId`, a user-facing read missing the published
   condition, an admin operation missing the role check, audience filtering done in a
   component. Also diff your files against their nearest siblings for convention drift.

3. **Phase 2 — backend behavior**, cases B1–B30, **against dev, never production**. B6
   (user B still sees what user A dismissed) and B8 (a new user gets no backlog) are the
   two that matter most; do not report a pass on either without observing it.

4. **Phase 3 — UI behavior.** Run the app and drive it. If you cannot drive a browser,
   say so explicitly and give the user the exact click-path to run themselves — do not
   silently mark UI cases as passed.

5. **Phase 4 — admin console**, including that a non-admin is refused both in the UI and
   when calling the operation directly.

6. **Fix and retest** anything that fails, unless `--no-fix`. Then re-run the affected
   cases.

7. **Report** with `assets/install-report.template.md`.

## Reporting rules

- ✓ only for results you observed yourself in this run.
- ✗ must say whether it was fixed and retested, or is left open and why.
- ○ (not run) must say why it could not be run.
- A partial pass with honest notes is the correct output. A green report that is wrong is
  worse than no report.
