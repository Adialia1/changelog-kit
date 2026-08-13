# 08 · Troubleshooting and unusual repos

## Symptom → cause

**Modal never appears.**
Walk the pipeline in order, stopping at the first false: (1) is there a published entry
with `highlight = true`? (2) is its `publishedAt` within `MODAL_LOOKBACK_DAYS` and not in
the future? (3) is `publishedAt > user.createdAt - GRACE`? — this catches out almost
everyone testing with their own long-lived admin account against a just-published entry,
which *should* work, and with a freshly seeded user against an older entry, which should
not. (4) does the audience match? (5) is there already a dismissal receipt? (6) is the
route suppressed / is auth still loading? (7) is the component actually mounted in the
authenticated layout — add a temporary log at mount.

**Modal reappears after dismissal.**
Almost always one of: the dismiss mutation is failing silently (check the network tab —
this is why the retry exists); the receipt is being written for the wrong user id; the
modal query is filtering on `seenAt` instead of `dismissedAt`; or two receipt rows exist
for the same (user, entry) pair because the upsert is not keyed on the composite index.

**Every user sees a modal on their first ever login.**
Invariant #5 is not implemented, or the user creation timestamp being compared is wrong
(e.g. comparing against the *session* creation time, which is always recent).

**One user's dismissal hides the entry for everyone.**
The receipt is not keyed by user, or the query result is being cached globally
(server-side cache without the user in the cache key, an ISR/static page, or a CDN
caching an authenticated response). Check cache headers on the endpoint.

**Draft visible to end users.**
A user-facing read is missing the status condition, or a shared helper is being reused by
both admin and user paths. Separate the projections.

**Targeted entry leaked to the wrong client.**
Audience filtering was done in the component. Move it server-side. Then check whether the
leaked payload was cached anywhere.

**Layout jumps when the modal opens.**
Scroll-lock is adding/removing a scrollbar. The repo's dialog primitive usually handles
this; if you hand-rolled the overlay, that is the bug.

**Screenshot looks fine locally, broken in production.**
A storage URL that only resolves in dev, a relative path that breaks under a different
base path, or an image committed to the public dir but not deployed.

**Gallery items appear in the wrong order.**
The query returns them in insertion order and nothing sorts by `order`, or `order` was
never renumbered after a removal and now has gaps that a stable sort resolves differently
than expected. Sort server-side, in the projection.

**The modal jumps or resizes when stepping through media.**
Intrinsic dimensions were not captured at upload, so each item sizes itself on load. Fix at
the source (record width/height on upload) and fall back to a fixed-ratio container for
items already stored without them.

**Clicking a per-item link records the wrong thing — or nothing.**
Usually the click handler passes the array index instead of the item id, so the data is
meaningless after any reorder, or it hardcodes `"cta"` for every target. Both look correct
on screen; only a database check catches them.

**Two videos play at once, or a video keeps playing after the user moves on.**
The element was reused across item changes instead of remounted. Key the video by item id,
and pause explicitly when the active item or entry changes.

**Video autoplay silently does nothing on iOS.**
Autoplay requires `muted` **and** `playsInline` together. Without `playsInline`, iOS opens
its fullscreen player instead — which is worse than not autoplaying at all.

**A GIF makes the modal crawl.**
GIFs are enormous and decode on the main thread. Convert to mp4/webm; the same clip is
typically 10× smaller. If conversion is not available, treat it as an image and accept the
cost, or drop it.

**One broken media item blanks the whole entry.**
The projection is returning a null url instead of dropping the item, and the component is
rendering into it. Filter unresolvable items server-side (`03-backend-operations.md`, U1).

---

## Repos that need a different shape

**No auth at all.** Per-user read state is impossible. Offer the reduced build: public
changelog page, plus a modal whose dismissal is stored in `localStorage` keyed by entry
slug. Be explicit that dismissal is per-browser, not per-person, and disappears when the
user clears storage. Do not silently downgrade — the user chooses.

**Static site / no database (Astro, Hugo, plain Next export).** Entries live as markdown
files in the repo and are read at build time. Dismissal is `localStorage`. There is no
admin console; "publishing" is merging a PR and deploying. The draft-then-publish gate is
still preserved — the PR review *is* the human gate. Skip references 03 and 05 entirely.

**Mobile app (React Native / Expo).** The backend contracts are unchanged. The modal is a
native modal or bottom sheet; the changelog page is a screen; route suppression becomes
navigation-state suppression. Media should be sized for the device and cached. Add: do not
show the modal while the app is resuming from background mid-task.

**Multi-tenant with per-tenant branding.** Entries are usually global, but a tenant may
want its own announcements. Two options: (a) `audience.mode = "tenants"` for global
entries with targeting — simplest; (b) an optional `tenantId` **owner** field on the entry
so a tenant admin can write entries only their own users see. (b) means the admin console
needs tenant scoping and the role check must distinguish platform admin from tenant admin.
Do not build (b) unless asked.

**Monorepo with several apps.** Decide early whether the changelog is shared across apps
or per app. If shared, the entry needs an `apps: string[]` field, filtered like audience,
and the shared package needs a home in the workspace. Record the decision in the
Adaptation Map.

**GraphQL API.** Operations become queries/mutations in the schema; the contracts and
invariants are unchanged. Take care that the entry type does not expose `audience` or
`status` fields to the public schema — split into `ChangelogEntry` and
`AdminChangelogEntry` types.

**No realtime layer.** The modal query runs once on mount; that is sufficient. Do not add
polling. A user who is mid-session when you publish sees it on their next navigation or
reload, which is fine.

---

## Performance notes

Only relevant at scale; do not pre-optimize.

- The modal query is the hot path: it runs once per authenticated session start. It reads
  a small indexed range plus one receipt lookup per candidate. That is fine into the
  hundreds of thousands of users.
- If receipt lookups become the cost, add the `changelog_user_state` cache
  (`02-data-model.md`, entity 3) and short-circuit when `unreadCount == 0`.
- Never iterate all users on publish. Receipts are created lazily, on view.
- Cap `MODAL_LOOKBACK_DAYS` — an unbounded range scan over years of entries is the one
  query here that can actually get slow.
- Entry counts are small (tens per year). Do not build a search index for them.

---

## Extending later

Things people ask for next, with the cheap version of each:

- **Email digest of the week's entries** — a scheduled job reading published entries in a
  date range through the repo's existing mailer. Do not build a second copy of the model.
- **Public marketing changelog page** — a separate unauthenticated operation returning
  only `audience.mode == "all"` entries. Never loosen the authenticated one.
- **RSS/JSON feed** — same source as the public page.
- **Per-entry reactions or comments** — a new table keyed by (user, entry); it does not
  belong in `receipts`, which is state, not feedback.
- **Scheduled publishing** — `publishedAt` in the future plus a query condition
  `publishedAt <= now()` (already specified), so all you need is a scheduler that flips
  status, or simply publish with a future date and let the condition do the work.
