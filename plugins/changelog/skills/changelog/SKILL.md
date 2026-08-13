---
name: changelog
description: >
  Install, extend, or operate a product changelog inside an existing web application:
  a "What's New" modal shown once per user, a changelog page, unread badges, per-user
  read state, and a draft-then-publish admin console. Contains a stack-agnostic
  specification (data model, backend operation contracts, UI contracts, security
  invariants, verification suite) that you adapt to the host repository's own database,
  auth system, routing, and design system — never copied in verbatim.
  Trigger on: "set up a changelog", "add what's new / release notes / product updates",
  "changelog modal", "notify clients about new features", or /changelog:install,
  /changelog:new, /changelog:verify.
---

# Changelog Kit

## What this skill is

A **specification, not a library**. It describes a product-changelog feature precisely
enough that you can build it correctly in an unfamiliar repository, using that
repository's existing database, auth, router, and component kit. There is no runtime
package to install and no code to copy. You read the contracts here, read the host
repo, and write code that looks like it was always part of that repo.

The feature has four surfaces:

| Surface | Who sees it | Purpose |
|---|---|---|
| **What's New modal** | End users, once per user per entry | Announce what shipped at login: several entries when several things shipped, each with a gallery of screenshots and short clips, and a link per item to the exact place in the app |
| **Changelog page** | End users, anytime | Full scrollable history of what shipped |
| **Unread badge** | End users | Small dot/count in nav so users know there is something new |
| **Admin console** | Staff/admin role only | Create drafts, upload media, target an audience, preview, **publish** |

The publishing model is **draft → human publish**. An agent (or a developer via
`/changelog:new`) creates a *draft*. Nothing reaches end users until a human presses
Publish in the admin console. This is deliberate: changelog copy is client-facing, and
an agent should never push text to a customer's screen unreviewed.

## Read this before you build

Do not start writing code from this file alone. Load the reference that matches the
phase you are in. Each is self-contained; read them in order for a fresh install.

| Phase | Reference | Load when |
|---|---|---|
| 1. Survey the repo | `references/01-discovery.md` | Always first on a new install. Produces the Adaptation Map. |
| 2. Data model | `references/02-data-model.md` | Before touching the database/schema. |
| 3. Backend operations | `references/03-backend-operations.md` | When writing queries/mutations/endpoints. |
| 4. User-facing UI | `references/04-ui-contracts.md` | When building modal, page, badge. |
| 5. Admin console | `references/05-admin-console.md` | When building the publish surface. |
| 6. Authoring entries | `references/06-authoring.md` | When running `/changelog:new` or writing entry copy. |
| 7. Verification | `references/07-verification.md` | After install, and before declaring done. **Not optional.** |
| 8. Problems | `references/08-troubleshooting.md` | When something behaves wrong, or the repo is unusual. |

Templates live in `assets/`:
- `assets/entry.template.md` — the on-disk form of one changelog entry.
- `assets/adaptation-map.template.md` — written into the host repo as
  `changelog/ADAPTATION-MAP.md`; records every decision you made so the *next* agent
  session does not re-derive them.
- `assets/install-report.template.md` — the completion report format.

## Non-negotiable invariants

These hold in every host repo, on every stack. If you cannot satisfy one, stop and say
so rather than shipping around it.

1. **Identity comes from the server.** Every read/write of per-user state derives the
   user id from the server-side session. A client-supplied `userId` argument is a
   vulnerability, not a convenience.
2. **Drafts are invisible.** No public/user-facing query may return an entry whose
   status is not `published`, under any argument combination.
3. **Audience filtering happens server-side.** Never ship all entries to the client and
   filter in the component — targeted entries can name customers, prices, or
   unreleased plans.
4. **Dismissal is per user, and permanent.** User A dismissing an entry must not affect
   user B. A dismissed entry never reappears for that user, across devices and sessions.
5. **No backfill blast.** A user never sees a modal for an entry published before their
   account existed. New signups get a clean first login.
6. **Publishing is a human action.** Agents create drafts. Humans publish.
7. **The modal is skippable and never blocks.** If the changelog backend errors or is
   slow, the app renders normally and the modal simply does not appear.
8. **No new runtime dependencies** unless the repo genuinely lacks a primitive (e.g. no
   dialog component at all). Prefer what is already there. A carousel is an index in
   state; it is not a reason to add a library.
9. **Every media item has alt text**, enforced at publish. In a gallery it is the only
   thing distinguishing item 2 from item 3 for a screen-reader user.

## Workflow

**Fresh install** (`/changelog:install`):
Discovery → write Adaptation Map → confirm plan with user → schema → backend ops →
admin console → user UI → seed a demo entry → run verification → report.

Build in that order. The admin console before the user UI is intentional: it gives you
a real way to create test data, and it is the surface that unblocks the human.

**New entry** (`/changelog:new`): interview → write entry file → upload media →
create draft in the target environment → hand the admin URL to the human.

**Verify** (`/changelog:verify`): run the derived test matrix in
`references/07-verification.md` against the actual install.

## Scope discipline

Build what is specified. Common over-reach to avoid: email digests, RSS feeds,
in-app tours, A/B testing of copy, per-entry comment threads, analytics dashboards.
Note them as possible follow-ups; do not build them unasked. The optional extras that
*are* in scope, and only when the user asks, are marked **[Optional]** in the
references.
