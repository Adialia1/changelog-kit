# Changelog · Adaptation Map

<!--
Written into the host repo as `changelog/ADAPTATION-MAP.md` during install.

Purpose: record every decision made when adapting the changelog spec to THIS repo, so a
future agent session (or a new contributor) does not re-derive them — or worse, derive
them differently. Update this file whenever the changelog implementation changes.

Replace every < > placeholder with a real path, symbol, or "none".
-->

Installed: <YYYY-MM-DD> · Spec version: changelog-kit 0.1.0

## Data layer

| Question | Answer |
|---|---|
| Database / ORM | <Convex / Prisma / Drizzle / Supabase / …> |
| Schema file | <path> |
| Naming convention | <camelCase / snake_case; plural / singular> |
| Id type | <platform id / uuid / int> |
| Timestamp convention | <epoch ms number / timestamptz / platform `_creationTime`> |
| Migration needed? | <yes — path / no, push-based> |
| Entry table name | <changelogEntries> |
| Receipt table name | <changelogReceipts> |
| User-state cache | <not implemented / table name> |

## Auth and identity

| Question | Answer |
|---|---|
| Server-side current user | `<symbol>` from `<path>` |
| Behavior when signed out | <returns null / throws> |
| User table + id type | <path, type> |
| User creation timestamp field | <field — or "added `changelogJoinedAt`"> |
| Admin/role check | `<symbol>` from `<path>` — role value: `<admin>` |
| Client auth hook | `<useX()>` from `<path>`; loading flag: `<isLoading>` |

## Tenancy

| Question | Answer |
|---|---|
| Multi-tenant? | <no / yes> |
| Tenant id field | <field or n/a> |
| Server-side tenant resolution | <symbol or n/a> |
| Audience modes enabled | <all / all+roles / all+roles+tenants+users> |

## Backend

| Question | Answer |
|---|---|
| Function location | <convex/changelog/ · src/server/changelog/ · …> |
| User-facing ops file | <path> |
| Admin ops file | <path> |
| Shared helpers file | <path> |
| Constants (MODAL_MAX_ENTRIES etc.) | <path> |
| Tests | <path — which cases from 07-verification are committed> |
| Rate limiting | <mechanism or "none — mitigated by 20-id cap"> |

## UI

| Question | Answer |
|---|---|
| Router | <Next App Router / Pages / React Router / …> |
| Authenticated layout (modal mount) | <path> |
| Component directory | <path> |
| Dialog primitive reused | `<Dialog>` from `<path>` |
| Gallery component | <path — shared by modal and changelog page> |
| Digest mode threshold | <stepper up to 3, digest 4–8> |
| Button / Badge / Card | <paths> |
| Styling system | <Tailwind v4 / v3 / CSS modules / …> |
| Token source | <path> |
| Dark mode mechanism | <next-themes / class on html / none> |
| Animation | <motion / CSS / none> |
| Markdown renderer | <symbol from path / plain text only> |
| i18n system | <none / symbol from path> |
| RTL required | <no / yes — locales> |
| Changelog page route | `</changelog>` at <path> |
| Deep link form | `</changelog/[slug]>` or `#slug` |
| Nav entry point | <path + which menu> |
| Unread badge | <implemented at path / skipped> |

## Media

| Question | Answer |
|---|---|
| Storage mechanism | <Convex file storage / S3 / public dir / …> |
| Upload operations | `<generateMediaUploadUrl / addMediaItem / reorderMedia / removeMediaItem>` at <path> |
| URL resolution | <how a stored ref becomes a public URL> |
| Media stored as | <related table `ChangelogMedia` / JSON column on the entry> |
| Video supported | <yes / no — reason> |
| Poster frames | <captured client-side / uploaded / none> |
| Size + duration limits | <5 MB image · 15 MB video · 60s> |
| Fallback for static assets | <public/changelog/ — requires deploy> |

## Admin console

| Question | Answer |
|---|---|
| Route | </admin/changelog> at <path> |
| Layout / guard reused | <path> |
| Live preview implemented | <yes / no> |
| Audience preview count | <yes / no — reason> |
| Engagement stats | <yes / no> |

## Publishing workflow

| Question | Answer |
|---|---|
| Entry files | `changelog/entries/YYYY-MM-DD-<slug>.md` |
| Sync command (dev) | `<command>` |
| Sync command (prod) | `<command>` |
| Duplicate-slug policy | <reject / suffix -2> |
| Admin console URL (prod) | <url> |

## Decisions and deviations

Record anything that differs from the spec, and why. Examples:

- `listEntries` marks visible entries seen but does not dismiss them — a user browsing
  history should still get the modal for anything unacknowledged.
- Tenant targeting omitted: single-tenant app.
- Media link clicks dismiss the entry (the alternative — mark seen only — was rejected
  because links here navigate in-app rather than opening docs in a new tab).
- `entryStats` omitted: receipt-count queries are expensive here without an extra index.

## Known gaps

- <e.g. no rate limiter in this repo; dismiss is protected only by the 20-id cap>
