# 02 · Data model

Three entities. Two are required; the third is a cache you add only if you measure a
need. Field names below are the *canonical* names — rename them to match the host repo's
casing convention from discovery, and keep the renaming consistent everywhere.

Table naming: prefix with the repo's domain convention if it has one. Default names:
`changelog_entries`, `changelog_receipts`, `changelog_user_state`. In a camelCase repo:
`changelogEntries`, `changelogReceipts`, `changelogUserState`.

---

## Entity 1 — Entry (required)

One shipped thing worth telling users about.

```
Entry {
  id            Id            -- platform primary key
  slug          string        -- unique, url-safe, stable: "bulk-csv-export"
                              -- used for deep links and for idempotent upserts from CLI

  title         string        -- <= 80 chars. The headline in modal and list.
  summary       string        -- <= 200 chars. Plain-language benefit. Shown in the modal
                              -- and as the list preview. Required.
  body          string?       -- Markdown. Full detail on the changelog page. Optional:
                              -- a small improvement may be summary-only.

  type          enum          -- "feature" | "improvement" | "fix" | "announcement"
  status        enum          -- "draft" | "published" | "archived"
  highlight     boolean       -- true  -> eligible for the What's New modal
                              -- false -> appears on the changelog page only ("quiet" entry)

  version       string?       -- optional release label shown as a chip: "v2.4"

  media         MediaItem[]   -- ordered gallery, 0..MEDIA_MAX_ITEMS. Empty array = no media.
  cta           Cta?          -- entry-level primary action; see below
  audience      Audience      -- see below; default { mode: "all" }

  publishedAt   timestamp?    -- null while draft. Set on publish. Sort key for users.
                              -- May be backdated by an admin; ordering follows this field.
  createdAt     timestamp
  updatedAt     timestamp
  createdBy     Id<User>
  publishedBy   Id<User>?
  archivedAt    timestamp?
}

MediaItem {
  id            string        -- stable, entry-local id (short uuid/nanoid). NOT the array
                              -- index: items get reordered, and click tracking must
                              -- survive that.
  order         number        -- explicit sort position, 0-based. Never rely on array
                              -- order alone once edits are possible.
  kind          enum          -- "image" | "video"
  ref           string        -- storage id OR absolute/relative URL, per repo storage
  alt           string        -- REQUIRED, non-empty. Enforced at publish, not just in
                              -- the form. Describes what the item SHOWS.
  caption       string?       -- <= 120 chars, rendered under the item. Optional, and
                              -- different from alt: the caption is for everyone, the alt
                              -- is for people who cannot see the item.
  link          Cta?          -- OPTIONAL per-item destination. "This screenshot shows the
                              -- export menu -> take me to the export menu."
  width         number?       -- intrinsic px; prevents layout shift
  height        number?
  posterRef     string?       -- video only: poster/first frame. Strongly recommended —
                              -- it is what fills the box before the video decodes.
  durationMs    number?       -- video only; drives the "0:12" chip and the autoplay rule
}

Cta {
  label         string        -- <= 24 chars, verb-first: "Try bulk export"
  href          string        -- internal path preferred: "/reports?export=1"
                              -- external must be absolute and opened in a new tab
}

Audience {
  mode          enum          -- "all" | "roles" | "tenants" | "users"
  roles         string[]?     -- when mode = "roles"
  tenantIds     Id[]?         -- when mode = "tenants"
  userIds       Id[]?         -- when mode = "users"
}
```

**Indexes** (name them per repo convention):
- `by_slug` on `[slug]` — unique lookup, idempotent upsert from the CLI.
- `by_status_publishedAt` on `[status, publishedAt]` — powers every user-facing read.
- `by_status_highlight_publishedAt` on `[status, highlight, publishedAt]` — powers the
  modal query. Merge with the previous index if the platform supports prefix reuse.
- `by_createdAt` on `[createdAt]` — admin list including drafts.

**Validation rules** (enforce server-side, in the publish operation — not only in a form):
- `slug` matches `^[a-z0-9]+(-[a-z0-9]+)*$` and is unique.
- `title` and `summary` non-empty after trim; length caps above.
- **Every** media item has non-empty `alt`. One item missing alt blocks the publish, and
  the error names which item ("Item 3 of 4 needs alt text").
- `media.length <= MEDIA_MAX_ITEMS` (6). Each item's `order` is unique within the entry.
- Media item ids are unique within the entry.
- A video item should have `posterRef`; warn if missing (it degrades to a blank box while
  loading) but do not block.
- If `cta` present, `label` non-empty and `href` non-empty; if `href` starts with `/`,
  it should resolve to a real route — check the router at authoring time and warn.
- Every media `link`, if present, follows the same `href` rules as `cta`. An item may have
  a link with no entry-level CTA, and vice versa.
- `audience.mode` other than `"all"` requires its matching array to be non-empty.
- Cannot publish an entry whose `status` is `archived` — un-archive first.

**Why `highlight` is separate from `type`.** Not everything published deserves an
interruption. A typo fix belongs on the page, not in a modal on a client's screen. The
`highlight` flag is the throttle, and the modal query filters on it. Default it to `false`
in the admin form for `type: "fix"`, `true` for `type: "feature"`.

**Why media is a list, and why each item carries its own link.** One shipped feature
usually has more than one thing to show — the new button, the dialog it opens, the result.
A single screenshot forces the author to either pick one or paste a collage. And once
there are several, a single entry-level CTA is the wrong shape: the screenshot of the
export menu should lead to the export menu, while the clip of the scheduled-report flow
should lead to scheduling. So:

- `entry.cta` — the entry's primary action, in the modal footer. At most one. Optional.
- `mediaItem.link` — "take me to *this*", rendered on the item itself. Any number, one per
  item. Optional per item.

Both are optional and independent. An entry can have three linked screenshots and no
footer CTA, or a footer CTA and no media at all.

**Two levels of "multiple".** Do not confuse them:

| Level | Means | Rendered as |
|---|---|---|
| Several entries pending | Several distinct things shipped | Stepper or digest — see `04-ui-contracts.md` |
| Several media in one entry | One thing, shown from several angles | A gallery inside that entry |

---

## Entity 2 — Receipt (required)

Per-user, per-entry read state. **This is the entity that makes dismissal per-user, which
is invariant #4.**

```
Receipt {
  id            Id
  userId        Id<User>      -- always taken from the server session
  entryId       Id<Entry>
  seenAt        timestamp?    -- the entry was actually rendered to this user
  dismissedAt   timestamp?    -- the user acknowledged (OK / Skip / Esc / CTA click)
  ctaClickedAt  timestamp?    -- optional; first click of ANY link in this entry
  clickedTargets string[]?    -- [Optional] which links were followed: "cta" for the
                              -- footer action, or a media item id. Capped at 10, deduped.
                              -- This is what answers "did the client actually click the
                              -- screenshot I pointed them at?"
}
```

**Indexes:**
- `by_user_entry` on `[userId, entryId]` — the hot path; treat as unique.
- `by_user` on `[userId]` — list a user's receipts for unread counting.
- `by_entry` on `[entryId]` — **[Optional]** per-entry engagement stats for the admin.

**Semantics:**
- Rows are created lazily. No receipt row = unseen. Never pre-create receipts for all
  users on publish; that is a write amplification bomb on a large user base.
- Writes are **upserts and idempotent**. Dismissing twice is a no-op, not an error.
- `dismissedAt` is the gate for the modal. `seenAt` alone does not suppress: a user who
  saw the modal but closed the tab mid-render should see it again. In practice you will
  stamp both together on dismissal; stamp `seenAt` on render.
- Deleting a user should cascade its receipts if the repo has a deletion routine — add
  it there (grep for an existing account-deletion path).

### Why receipts instead of a single "lastSeen" watermark

A watermark (`user.changelogLastSeenAt`) is one field and O(1) to check. It is also
wrong in three ways this feature actually hits:

1. **Backdated publishes.** An admin publishes with `publishedAt` set to last Tuesday.
   Every user whose watermark is newer silently never sees it.
2. **Partial dismissal.** Three entries are queued; the user acknowledges one and closes
   the tab. A watermark either loses the other two or re-shows the acknowledged one.
3. **No per-entry signal.** You cannot answer "did anyone click the CTA?" — which is the
   whole reason the developer added the CTA.

Receipts cost one small row per (user, entry) that a user actually sees. Bounded by
`users × highlighted entries`, and only for entries genuinely displayed.

---

## Entity 3 — User state cache (optional)

Only add this if the unread-count query shows up as a real cost. It is a denormalization,
and denormalizations drift.

```
UserState {
  userId          Id<User>    -- unique
  lastCheckedAt   timestamp   -- last time the user opened the changelog page
  unreadCount     number      -- cached; recomputed on publish and on dismiss
  snoozedUntil    timestamp?  -- [Optional] "remind me later" support
}
```

If you add it, the invariant is: `unreadCount` is a cache, never the source of truth. The
badge may read it; the modal query must not.

---

## Invariant #5 — the no-backfill rule

A user must never be shown a modal for an entry published before their account existed.

```
eligible(entry, user) requires entry.publishedAt > user.createdAt
```

Discovery told you the user's creation timestamp field. Three cases:

- **Field exists** → use it directly.
- **Platform-managed creation time** (e.g. Convex `_creationTime`) → use it.
- **No creation time anywhere** → add `changelogJoinedAt` to the user record, set it on
  first authenticated changelog query when absent, and use that. Existing users get
  stamped on their next visit, which correctly means "no historical backlog for anyone
  who is already here."

Add a small grace window so a user who signs up seconds before a publish is not skipped
by clock skew:

```
GRACE = 5 minutes
eligible if entry.publishedAt > (user.createdAt - GRACE)
```

---

## Localized copy [Optional]

Only if discovery found an i18n system **and** the product's users span locales.

Two workable shapes; pick per repo:

- **Per-locale fields**: `title: { en: string, he: string }`. Simple, but every locale
  must be filled or fall back at read time. Store a `defaultLocale` and fall back to it.
- **Sibling rows**: one entry row per locale sharing a `slug` plus a `locale` field, with
  the reader filtering on the user's locale and falling back. More rows, cleaner queries,
  easier partial translation.

For a single-locale product, store plain strings. Do not build translation plumbing
speculatively — but *do* put every static UI label (button text, headings, empty states)
through the repo's i18n system regardless, since that is nearly free.

---

## Worked mappings

The same model in three idioms. These are illustrations of shape, not code to paste —
match the host repo's actual conventions, imports, and helper functions.

### Convex

```ts
// convex/schema.ts — added to the existing defineSchema({...}) object
changelogEntries: defineTable({
  slug: v.string(),
  title: v.string(),
  summary: v.string(),
  body: v.optional(v.string()),
  type: v.union(v.literal("feature"), v.literal("improvement"),
                v.literal("fix"), v.literal("announcement")),
  status: v.union(v.literal("draft"), v.literal("published"), v.literal("archived")),
  highlight: v.boolean(),
  version: v.optional(v.string()),
  media: v.array(v.object({
    id: v.string(),                            // entry-local stable id
    order: v.number(),
    kind: v.union(v.literal("image"), v.literal("video")),
    storageId: v.optional(v.id("_storage")),   // uploaded via Convex file storage
    url: v.optional(v.string()),               // or an external/static URL
    alt: v.string(),
    caption: v.optional(v.string()),
    link: v.optional(v.object({ label: v.string(), href: v.string() })),
    width: v.optional(v.number()),
    height: v.optional(v.number()),
    posterStorageId: v.optional(v.id("_storage")),
    posterUrl: v.optional(v.string()),
    durationMs: v.optional(v.number()),
  })),                                         // default [] — not optional; an empty
                                               // array is simpler to read than undefined
  cta: v.optional(v.object({ label: v.string(), href: v.string() })),
  audience: v.object({
    mode: v.union(v.literal("all"), v.literal("roles"),
                  v.literal("tenants"), v.literal("users")),
    roles: v.optional(v.array(v.string())),
    tenantIds: v.optional(v.array(v.string())),
    userIds: v.optional(v.array(v.id("users"))),
  }),
  publishedAt: v.optional(v.number()),
  updatedAt: v.number(),
  createdBy: v.id("users"),
  publishedBy: v.optional(v.id("users")),
  archivedAt: v.optional(v.number()),
})
  .index("by_slug", ["slug"])
  .index("by_status_published", ["status", "publishedAt"])
  .index("by_status_highlight", ["status", "highlight", "publishedAt"]),

changelogReceipts: defineTable({
  userId: v.id("users"),
  entryId: v.id("changelogEntries"),
  seenAt: v.optional(v.number()),
  dismissedAt: v.optional(v.number()),
  ctaClickedAt: v.optional(v.number()),
})
  .index("by_user_entry", ["userId", "entryId"])
  .index("by_user", ["userId"]),
```

Notes: `createdAt` is covered by `_creationTime`; do not duplicate it. Convex queries must
go through an index — `.withIndex(...)`, not `.filter(...)` on a large table. Storage ids
are resolved to URLs server-side with `ctx.storage.getUrl(...)` in the projection — do one
`Promise.all` over the media array rather than a sequential await per item.

### SQL (Prisma-flavored)

```prisma
model ChangelogEntry {
  id          String    @id @default(cuid())
  slug        String    @unique
  title       String
  summary     String
  body        String?
  type        String
  status      String    @default("draft")
  highlight   Boolean   @default(false)
  version     String?
  ctaLabel    String?
  ctaHref     String?
  audience    Json      // { mode, roles?, tenantIds?, userIds? }
  publishedAt DateTime?
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  createdById String
  publishedById String?
  archivedAt  DateTime?
  media       ChangelogMedia[]
  receipts    ChangelogReceipt[]

  @@index([status, publishedAt])
  @@index([status, highlight, publishedAt])
}

model ChangelogMedia {
  id         String   @id @default(cuid())
  entryId    String
  entry      ChangelogEntry @relation(fields: [entryId], references: [id], onDelete: Cascade)
  order      Int
  kind       String   // "image" | "video"
  ref        String
  alt        String
  caption    String?
  linkLabel  String?
  linkHref   String?
  width      Int?
  height     Int?
  posterRef  String?
  durationMs Int?

  @@index([entryId, order])
}

model ChangelogReceipt {
  id           String    @id @default(cuid())
  userId       String
  entryId      String
  entry        ChangelogEntry @relation(fields: [entryId], references: [id], onDelete: Cascade)
  seenAt       DateTime?
  dismissedAt  DateTime?
  ctaClickedAt DateTime?

  @@unique([userId, entryId])
  @@index([userId])
}
```

A separate `ChangelogMedia` table versus a `Json` column on the entry: both work. Take the
table when the repo's convention is relational and normalized (you get ordering, cascade
delete, and per-item queries for click stats). Take a `Json media` column when the repo
already stores structured blobs that way and you want one round trip. Media is only ever
read as a whole entry, so the join is not a performance argument either way — pick the one
that matches the repo and record the choice in the Adaptation Map.

If the repo is Supabase or otherwise relies on Row Level Security, RLS is **additional**
to the checks in `03-backend-operations.md`, not a replacement: entries readable only when
`status = 'published'`; receipts readable/writable only where `user_id = auth.uid()`;
writes to entries restricted to the admin role.

### Document store (Firestore-flavored)

```
/changelogEntries/{entryId}       -- fields as in the canonical model
/changelogReceipts/{userId}_{entryId}
      -- deterministic composite doc id makes the upsert a single set() with merge,
      -- and makes the security rule trivial: request.auth.uid == resource id prefix
```

Composite ids are the idiomatic way to get the (user, entry) uniqueness constraint
without a transaction.
