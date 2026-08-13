# 01 · Discovery — reading the host repository

Goal: end this phase able to answer every row of the **Adaptation Map** with a concrete
file path or symbol name. Everything downstream is a translation of the spec into the
answers you collect here. Guessing here produces code that compiles and feels foreign.

Budget roughly 10–20 tool calls. Do not read whole large files; grep for the symbol,
then read the 30 lines around it.

---

## 1. Data layer

Identify how this repo defines and queries persistent data.

```
Detect by file:
  convex/schema.ts               -> Convex
  prisma/schema.prisma           -> Prisma (find the client singleton too)
  drizzle.config.*, */schema.ts  -> Drizzle
  supabase/migrations/*.sql      -> Supabase (note: RLS policies matter, see below)
  models/*.py, alembic/          -> SQLAlchemy
  app/models/*.rb, db/schema.rb  -> ActiveRecord
  *.entity.ts + nestjs           -> TypeORM
  firebase.json, firestore.rules -> Firestore
```

Record:
- **Where tables/collections are declared** (exact file).
- **Naming convention**: `snake_case` vs `camelCase`; plural vs singular table names;
  does the repo prefix domain tables (`app_users`, `crm_contacts`)? Match it exactly.
- **Id type**: opaque platform id, uuid, autoincrement int, string slug.
- **Timestamps**: does the repo use `createdAt: number` (epoch ms), `created_at`
  timestamptz, or platform-managed `_creationTime`? Match it. Do not introduce a second
  time convention.
- **Migration mechanism**: is a migration file required for new tables, or is the schema
  push-based? If migrations are required, you must write one.
- **Soft delete / archival convention**, if any.

> Convex specifics: tables are declared in `convex/schema.ts` with `defineTable`;
> `_id` and `_creationTime` are automatic; indexes are declared via `.index("by_x", ["x"])`
> and **queries must use an index** rather than filtering large tables. Check whether the
> repo groups functions into subdirectories (`convex/members/`, `convex/orders/`) and
> follow that grouping.

## 2. Auth and identity

This is the highest-risk area. Find the canonical server-side "who is calling" helper —
do not invent your own.

```
Grep for, in order:
  getAuthUserId | getCurrentUser | requireUser | requireAuth | currentUser
  ctx.auth.getUserIdentity | auth() | getSession | getServerSession
  betterAuth | @clerk/ | next-auth | supabase.auth.getUser
```

Record:
- **Server-side accessor** for the current user, with its exact import path and return
  shape (does it return `null` or throw when signed out?).
- **The user table** and its primary key type — receipts will reference it.
- **User creation timestamp field** — required for invariant #5 (no backfill blast).
  If the user record has no creation time, note it; the fallback is in `02-data-model.md`.
- **Role / permission check** used to gate admin surfaces. Grep for `role`, `isAdmin`,
  `permissions`, `staff`, `owner`. Find how an *existing* admin-only route or mutation
  guards itself and copy that mechanism verbatim.
- **Client-side auth state** hook for the UI (`useUser`, `useSession`, `useAuth`,
  `useConvexAuth`) and how a component knows auth is still loading. The modal must not
  render or query while auth is unresolved.

## 3. Tenancy

Multi-tenant repos need audience targeting by tenant; single-tenant repos do not.

```
Signals: tenants/ or orgs/ directory, tenantId/orgId/workspaceId/companyId columns,
subdomain routing in middleware, a tenant switcher component, RLS policies keyed on org.
```

Record: the tenant id field name, how the current tenant is resolved server-side, and
whether a user can belong to several tenants. If the app is single-tenant, write "none"
and skip tenant targeting entirely — do not build unused targeting UI.

## 4. Routing and app shell

- **Router**: Next.js App Router (`app/`), Next.js Pages (`pages/`), React Router,
  Remix, SvelteKit, Nuxt, Vue Router, Rails views, Django templates.
- **Where authenticated pages live**: e.g. `app/(dashboard)/`, `src/app/app/`,
  `apps/web/app/`. New pages must land in the same group so they inherit the layout.
- **The single authenticated layout component** that wraps every signed-in page. This is
  where the modal mounts — find it, do not create a new provider tree.
- **Admin route convention**: is there already `/admin/...`? Which layout guards it?
- **Server vs client component conventions** (App Router): note whether pages are server
  components that pass data down, or client components that fetch. Match it.

## 5. Design system

The output has to look native. Collect actual component paths, not vibes.

- **Component kit**: shadcn/ui (`components.json` present, `components/ui/*`), MUI,
  Chakra, Mantine, Ant, Radix primitives directly, or bespoke.
- **The existing modal/dialog component** — exact path and API (`<Dialog open onOpenChange>`
  vs `<Modal isOpen onClose>`). **Never introduce a second dialog implementation.**
- **Button, Badge, Card, ScrollArea, Skeleton** equivalents.
- **Styling**: Tailwind (which version? v4 uses `@theme` in CSS, v3 uses
  `tailwind.config`), CSS modules, styled-components, vanilla-extract.
- **Design tokens**: find the color/spacing/radius source of truth. Grep the CSS for
  `--background`, `--foreground`, `--primary`, `--radius`, or the theme object. Use
  token names, never raw hex values.
- **Dark mode**: `next-themes`? a `dark` class on `<html>`? Both themes must be handled.
- **Animation library** already present (`framer-motion` / `motion`, CSS transitions).
  Use it, or use none. Do not add one.
- **Typography scale** and how headings are usually written in this repo.

## 6. Localization and direction

Easy to miss, expensive to retrofit.

- Is there an i18n system (`next-intl`, `i18next`, a `messages/` or `locales/` dir, or a
  hand-rolled dictionary)? If so, **all new UI strings go through it** and the entry
  content model may need per-locale fields (see `02-data-model.md`, "Localized copy").
- Is any locale RTL (Hebrew, Arabic)? Check for `dir="rtl"`, `rtl:` Tailwind variants, or
  logical properties. If yes: use logical CSS properties (`margin-inline-start`, not
  `margin-left`), and mirror any directional icon (arrows in the carousel).
- Date formatting convention already in use — reuse it for entry dates.

## 7. File storage

For screenshots in entries.

- Existing upload path? Grep for `generateUploadUrl`, `storage.store`, `s3`, `putObject`,
  `cloudinary`, `uploadthing`, `supabase.storage`, `multer`.
- If one exists, reuse it and note how a stored file is turned back into a public URL.
- If none exists, the fallback is static assets under the public directory — note the
  path and that new images require a redeploy.

## 8. Environments and deployment

- How does one run a backend function against **production** vs **dev** from the CLI?
  (`npx convex run --prod`, `psql $DATABASE_URL`, `rails runner`, a `scripts/` helper.)
  `/changelog:new` needs this to create the draft in the right environment.
- Where do env vars live and how are they named?
- Is there a seed script convention you should extend rather than duplicate?

## 9. Conventions and quality gates

- Test framework and where tests live; is there an existing pattern for backend tests?
- Lint/typecheck/format commands (read `package.json` scripts / Makefile / justfile).
  You will run these in verification.
- Read `CLAUDE.md` / `AGENTS.md` / `CONTRIBUTING.md` if present — they outrank this
  skill's stylistic preferences for anything repo-specific.
- Commit conventions, if you will be committing.

---

## Output of this phase

Write `changelog/ADAPTATION-MAP.md` in the host repo from
`assets/adaptation-map.template.md`, filled in with real paths. Then present a short
plan to the user covering: where the schema goes, where the functions go, which routes
you will add, which existing components you will reuse, and anything you could not
determine. **Get confirmation before writing feature code.**

Two questions are worth asking the user explicitly, because the repo cannot answer them:

1. **Who is the audience?** End customers of the product, or internal staff? This
   changes the copy voice and whether targeting is needed.
2. **Which role publishes?** Name the exact role value that gates the admin console.

If discovery finds something genuinely incompatible — no auth system at all, no way to
identify a user — say so plainly. A changelog with per-user read state requires user
identity; without it, offer the reduced build (public changelog page + localStorage
dismissal) and let the user choose.
