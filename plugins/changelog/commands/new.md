---
description: Draft a changelog entry for what you just shipped — interview, optional screenshot, then create a draft for a human to publish
argument-hint: '[--env dev|prod] [--no-screenshot] [--since <ref>] [what shipped...]'
disable-model-invocation: true
---

Create a changelog **draft** for work that just shipped. A human publishes it; you never do.

Raw arguments: `$ARGUMENTS`

Argument handling:
- `--env dev|prod` — which environment gets the draft. Default: ask, and state the
  deployment name out loud before writing to production.
- `--no-screenshot` — skip media entirely.
- `--since <ref>` — diff base for working out what shipped. Default: the commit of the
  most recent file in `changelog/entries/`, else the last tag, else the last 20 commits.
- Free text is the user's own description of the feature — the strongest signal available;
  prefer it over anything you infer from git.

## Procedure

Load the `changelog` skill, then follow `references/06-authoring.md`.

1. **Read `changelog/ADAPTATION-MAP.md`.** It tells you the sync command, the environments,
   the storage mechanism, and the admin URL. If it is missing, the changelog is not
   installed here — offer `/changelog:install` and stop.

2. **Work out what shipped before asking anything.** Git log/diff since the base, plus this
   session's own context if you built it. Skip changes that are not user-visible (refactors,
   CI, deps) and say so rather than manufacturing an announcement.

3. **Propose a complete draft** — title, type, summary, CTA, popup-or-page — and ask only
   what you genuinely cannot infer, in one batch: screenshot, audience, and anything the
   client should know that the code does not say.

4. **Media** — capture it with browser automation if the app runs and the feature is
   reachable; otherwise accept a file path; otherwise skip. Write the alt text yourself and
   show it for correction. Record intrinsic width/height.

5. **Write** `changelog/entries/YYYY-MM-DD-<slug>.md` from `assets/entry.template.md`, with
   the media file alongside.

6. **Create the draft** in the chosen environment using the sync command from the Adaptation
   Map. Confirm the environment out loud first. If the slug is already published, do not
   overwrite — report the conflict and offer a new slug.

7. **Hand off**: file path, environment, draft status, and the admin console URL where the
   human reviews and publishes.

## Voice

Write for the customer, not the team. Benefit first, second person, ≤2 sentences of
summary, concrete numbers over adjectives, name the surface ("on the Reports page"), no
ticket numbers or component names, no hype. The full guide and before/after examples are
at the end of `references/06-authoring.md` — read it before writing copy.

## Rules

- The result is always a **draft**. Never publish, never flip status, never bypass the
  admin console.
- One entry per user-visible change. Do not bundle unrelated work into "various improvements".
- Do not commit unless asked. Never push.
- If the sync fails, leave the entry file in place and print the exact command for the user
  to run themselves.
