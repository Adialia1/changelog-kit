# Changelog Kit

**A Claude Code plugin that adds a product changelog to an app you already have** — a
"What's New" modal each user sees once, a changelog page, unread badges, and a
draft-then-publish admin console.

It ships **no library and no runtime dependency**. What it ships is a specification —
data model, backend operation contracts, UI contracts, security invariants, and a
verification suite — precise enough that a coding agent can read your repository and build
the feature *in your stack, with your components, following your conventions*.

---

## The problem

You ship constantly. Your client doesn't see any of it.

They log in, the app looks the same as last week, and eventually you get the message:
*"I don't really know what you've been working on."* Meanwhile the work is all there — in
commits, in Linear, in a Slack thread they stopped reading in March.

The usual fixes don't hold. A `CHANGELOG.md` in the repo is written for developers and
nobody outside the team ever opens it. A monthly email gets archived unread. A Notion page
goes stale in three weeks. None of them are *in the product*, at the moment the client is
already looking at it.

## The goal

**Close the loop between shipping a feature and the client knowing it exists.**

Concretely, when you finish a feature:

1. You describe it once, in your editor, in plain language.
2. It becomes a draft with screenshots or a short clip, each linked to the exact place in
   the app where the feature lives.
3. A human reviews the copy and publishes.
4. The next time each client logs in, they see it once — with a button that takes them
   straight there.
5. They click "Got it" and never see it again. Their colleague still sees it, once, on
   their own next login.
6. It stays on a changelog page they can scroll any time.

And the second goal, which is why this is a plugin and not a library: **it should work in
the app you already have**, not require you to adopt a new stack, a new component kit, or
a third-party widget with someone else's branding in it.

---

## How it works

Three commands, one loop.

```
                    ┌──────────────────────────────────────────────┐
   ONE TIME         │  /changelog:install                          │
   per repo         │                                              │
                    │  reads your repo → writes ADAPTATION-MAP.md  │
                    │  → shows you a plan → builds → verifies      │
                    └──────────────────────────────────────────────┘
                                        │
                                        ▼
   EVERY TIME       ┌──────────────────────────────────────────────┐
   you ship         │  /changelog:new                              │
                    │                                              │
                    │  reads git + this session → proposes copy    │
                    │  → captures screenshots/clips → writes       │
                    │  changelog/entries/*.md → creates a DRAFT    │
                    └──────────────────────────────────────────────┘
                                        │
                                        ▼
   HUMAN GATE       ┌──────────────────────────────────────────────┐
                    │  /admin/changelog  (in your own app)         │
                    │                                              │
                    │  edit copy · reorder media · pick audience   │
                    │  · live preview · PUBLISH                    │
                    └──────────────────────────────────────────────┘
                                        │
                                        ▼
   YOUR CLIENTS     ┌──────────────────────────────────────────────┐
                    │  next login → modal, once, per person        │
                    │  "Got it" → never again for them             │
                    │  /changelog → the full history, any time     │
                    └──────────────────────────────────────────────┘
```

### 1. Install — once per repository

```bash
cd ~/your-app
/changelog:install
```

The agent surveys your repo before writing anything: where the schema lives and how tables
are named, which auth system tells the server who is calling, which role gates admin pages,
whether the app is multi-tenant, which dialog component already exists, which design tokens
you use, whether any locale is RTL, how files get uploaded, and how to run a backend
function against dev versus production.

It writes all of that down in `changelog/ADAPTATION-MAP.md`, shows you a plan, and **waits
for your go-ahead** before touching feature code. Then it builds: schema, backend
operations, admin console, then the user-facing UI — using your dialog, your buttons, your
tokens, your file layout. Then it runs a test suite it derives from what it just built, and
reports what passed, what it fixed, and what it could not verify.

Adding no runtime dependency is a hard rule. If your repo has a `Dialog`, that is the
dialog. A carousel is an index in state, not a reason to install a library.

### 2. Ship a feature, then describe it — every time

```bash
/changelog:new
```

The agent works out what shipped from your git history and the session you just had, then
*proposes* the entry rather than interviewing you:

> Looks like you shipped bulk CSV export on the reports page. Draft:
>
> **Bulk CSV export** — *feature*
> "Export any report to CSV, including filtered views. Up to 50,000 rows per file."
>
> I'd capture three: the Export button (→ `/reports`), the export dialog
> (→ `/reports?export=1`), and the result. Who should see it?

You correct it. It captures the screenshots by driving the running app, writes
`changelog/entries/2026-08-13-bulk-csv-export.md`, and creates a **draft** in the
environment you name.

There is a voice guide behind this, because it is the part that decides whether any of it
is worth reading. "Matches are now ranked by how well they fit your criteria" — not
"Refactored the matching engine (#442)".

### 3. A human publishes

The agent's job ends at a well-formed draft plus a link. In your own app's admin console
you edit the copy, reorder the media, pick who sees it, preview the exact modal your client
will get — in light and dark — and press Publish.

This gate is deliberate. Changelog copy lands on a paying customer's screen, and
unpublishing does not un-see it.

### 4. Clients find out

On their next login, each user sees the modal once. Dismissal is **per user**: five people
in one client account each get it, and each "Got it" sticks for that person only, across
devices and sessions. Ship several things at once and they get a stepper of up to three, or
a scannable digest beyond that. Everything lives on the changelog page afterwards, and an
unread dot in the nav tells them when something is new.

---

## What gets built

| Surface | Who | What it does |
|---|---|---|
| **What's New modal** | Each user, once per entry | Announces what shipped at login — headline, plain-language summary, a gallery of screenshots and short clips, and a button per item that takes them straight there |
| **Changelog page** | Users, anytime | The full history, newest first, with deep links per entry |
| **Unread badge** | Users | A quiet dot in the nav when something is new |
| **Admin console** | Staff only | Write drafts, upload media, reorder, target an audience, preview, publish |

Each entry carries an ordered gallery of up to six images and videos, and **each item can
link somewhere of its own** — the screenshot of the export menu leads to the export menu,
the clip of scheduling leads to scheduling.

---

## Install the plugin

```bash
# in Claude Code
/plugin marketplace add Adialia1/changelog-kit
/plugin install changelog@changelog-kit
```

Or from a local clone:

```bash
git clone https://github.com/Adialia1/changelog-kit.git
# then, in Claude Code
/plugin marketplace add ~/changelog-kit
/plugin install changelog@changelog-kit
```

Then, in the repo you want the changelog in:

```bash
/changelog:install --dry-run   # survey + plan + Adaptation Map, no code written
/changelog:install             # the real thing
/changelog:new                 # after each feature ships
/changelog:verify              # re-run the derived test suite anytime
```

`--dry-run` is the cheapest way to see whether the survey read your repo correctly before
it touches a single file.

---

## Design principles

**Specification, not scaffolding.** Copied code ages badly and always looks foreign. The
plugin describes *what must be true* — the operations, their semantics, the security
invariants, the UI states — and the agent expresses that in your idiom: a Convex mutation,
a Prisma route handler, a Supabase RPC, a Rails action.

**Invariants over implementation.** Nine rules hold on every stack. Identity always comes
from the server. Drafts are never visible to users. Audience filtering never happens in the
browser. One user's dismissal never affects another. New signups never get blasted with a
backlog of old announcements — the modal only shows entries published after their account
existed. Publishing is a human action. A backend failure renders nothing rather than
breaking the app.

**Verification is derived, not scripted.** The plugin cannot know your test runner, so it
tells the agent how to *derive* the matrix from what it just installed — 30 backend cases,
23 UI cases, 11 admin cases — run them, fix what fails, and report honestly. Anything not
actually executed is reported as not run, never as passed.

**The install writes down its own decisions.** `changelog/ADAPTATION-MAP.md` records every
choice: which dialog was reused, which role publishes, which command pushes to production.
The next session reads it instead of guessing again.

---

## Works with

Any web stack with user accounts. Reference mappings cover Convex, SQL via Prisma/Drizzle,
Supabase, and document stores; the contracts are identical everywhere. The troubleshooting
reference covers the awkward cases — no auth at all, static sites with no database, React
Native, multi-tenant apps, monorepos, GraphQL.

## What's in the box

```
plugins/changelog/
  commands/
    install.md            /changelog:install
    new.md                /changelog:new
    verify.md             /changelog:verify
  skills/changelog/
    SKILL.md              router + the nine invariants
    references/
      01-discovery.md          how to read the host repo
      02-data-model.md         entities, indexes, worked mappings
      03-backend-operations.md 12 operation contracts + security checklist
      04-ui-contracts.md       modal, gallery, page, badge — states, a11y, adaptation
      05-admin-console.md      the publish surface
      06-authoring.md          interview flow + copy voice guide
      07-verification.md       the derived test suite
      08-troubleshooting.md    symptoms, unusual repos, extensions
    assets/
      entry.template.md
      adaptation-map.template.md
      install-report.template.md
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). The most useful contributions are worked mappings
for stacks not yet covered, and troubleshooting entries for repo shapes that needed a
different approach.

## License

MIT — see [LICENSE](LICENSE).
