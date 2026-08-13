# 03 · Backend operation contracts

Twelve operations: six for end users, six for admins. Each is specified as a contract —
name, auth requirement, arguments, return shape, and behavior. Implement them in the host
repo's idiom (Convex query/mutation, tRPC procedure, REST route handler, GraphQL
resolver, Rails controller action) and in the host repo's file layout.

Pseudocode below is deliberately not any one language. Translate the *semantics*.

Naming: use the repo's convention. If functions are grouped by domain, create the group
(`convex/changelog/`, `src/server/changelog/`, `app/services/changelog/`).

---

## Shared helpers

Write these once; every operation below leans on them.

```
requireUser(ctx) -> User
    user = <repo's canonical server-side session accessor>
    if !user: throw Unauthorized
    return user

requireAdmin(ctx) -> User
    user = requireUser(ctx)
    if !<repo's existing admin/role check>(user): throw Forbidden
    return user
    -- Copy the mechanism from an existing admin-guarded function in this repo.
    -- Do not invent a new role system.

matchesAudience(entry, user) -> boolean
    switch entry.audience.mode:
      "all"     -> true
      "roles"   -> entry.audience.roles   contains user.role
                   (or intersects user.roles, if the repo has multi-role users)
      "tenants" -> entry.audience.tenantIds contains <user's current tenant id>
      "users"   -> entry.audience.userIds contains user.id
    -- Evaluated on the server, always. Never send a non-"all" entry to a client that
    -- does not match; the audience list itself is sensitive.

isEligibleForModal(entry, user, receiptsByEntryId) -> boolean
    entry.status == "published"
    AND entry.highlight == true
    AND entry.publishedAt != null AND entry.publishedAt <= now()
    AND entry.publishedAt > (user.createdAt - GRACE_MS)      -- invariant #5
    AND matchesAudience(entry, user)
    AND receiptsByEntryId[entry.id]?.dismissedAt == null     -- invariant #4
```

`GRACE_MS = 5 * 60 * 1000`. Define it as a named constant in one place with the other
tunables:

```
MODAL_STEPPER_MAX   = 3       -- up to this many pending entries -> one-at-a-time stepper
MODAL_MAX_ENTRIES   = 8       -- hard cap per session; beyond MODAL_STEPPER_MAX the modal
                              -- switches to digest mode (see 04-ui-contracts.md).
                              -- Anything past this cap stays on the changelog page.
MODAL_LOOKBACK_DAYS = 60      -- ignore highlights older than this even if never dismissed
MEDIA_MAX_ITEMS     = 6       -- media items per entry
PAGE_SIZE           = 20
GRACE_MS            = 5 min
```

`MODAL_LOOKBACK_DAYS` matters: a user who is away for six months should get a short,
current welcome-back, not a nine-item slideshow. Older unseen entries remain on the
changelog page.

---

## User-facing operations

### U1 · `getPendingHighlights` — read

The modal's only data source.

```
auth:    required
args:    none
returns: Entry[]  (public projection; see "Projection" below), newest first, <= MODAL_MAX_ENTRIES

behavior:
    user = requireUser(ctx)
    cutoff = now() - MODAL_LOOKBACK_DAYS
    candidates = index(by_status_highlight_publishedAt)
                   .range(status = "published", highlight = true, publishedAt >= cutoff)
                   .order(desc)
                   .take(MODAL_MAX_ENTRIES * 4)      -- overfetch: audience + receipts prune
    receipts   = receipts.by_user_entry for (user.id, candidates ids)
    eligible   = candidates.filter(e => isEligibleForModal(e, user, receipts))
    return eligible.slice(0, MODAL_MAX_ENTRIES).map(publicProjection)
```

Notes:
- Overfetch factor 4 is a heuristic; if audiences are heavily targeted, raise it or loop
  until the take is satisfied or the range is exhausted.
- Must never throw for an ordinary signed-out client — the UI calls this behind an auth
  gate, but if it can be called while signed out, return `[]` rather than raising, so a
  race during sign-in does not surface an error toast.
- **Resolve every media item's storage ref to a URL here, server-side**, including video
  poster frames. The client receives ready-to-render URLs and never sees a storage id.
  Resolve the whole gallery concurrently (one `Promise.all` over the array), not in a loop
  — with 3 entries × 6 items that is 24 sequential round trips if you get it wrong.
- Return media **sorted by `order`**. Do not make the client sort.
- If a media item's ref fails to resolve (deleted file, expired URL), drop that item from
  the array rather than returning a null url. One broken screenshot must not blank the
  whole announcement.

### U2 · `listEntries` — read

The changelog page.

```
auth:    required (or public — see below)
args:    { cursor?: string, limit?: number = PAGE_SIZE, type?: EntryType }
returns: { entries: Entry[], nextCursor: string | null }

behavior:
    user = requireUser(ctx)
    page = index(by_status_publishedAt)
             .range(status = "published", publishedAt <= now())
             .order(desc).paginate(cursor, limit)
    return { entries: page.items.filter(matchesAudience(_, user)).map(publicProjection),
             nextCursor: page.next }
```

Audience filtering after pagination can yield short pages. Acceptable — the client
paginates until `nextCursor` is null. If most entries are targeted, filter before
paginating instead.

**Public changelog variant [Optional]:** if the user wants an unauthenticated marketing
changelog, add a separate operation that returns only entries with `audience.mode = "all"`
and never touches receipts. Keep it a *separate function* — do not make U2's auth
optional, because that is exactly how targeted entries leak.

### U3 · `getEntryBySlug` — read

Deep links (`/changelog/bulk-csv-export`) and CTA landing.

```
auth:    required
args:    { slug: string }
returns: Entry | null
behavior:
    entry = index(by_slug).get(slug)
    if !entry or entry.status != "published" or !matchesAudience(entry, user): return null
    -- Return null, not 403: a 403 confirms the entry exists to someone not targeted.
```

### U4 · `markSeen` — write

```
auth:    required
args:    { entryIds: Id[] }   -- max 20, reject longer
returns: void
behavior:
    user = requireUser(ctx)
    for id in entryIds:
        verify entry exists AND is published AND matchesAudience  -- do not let a client
                                                                  -- create receipts for
                                                                  -- arbitrary ids
        upsert receipt(user.id, id): set seenAt = now() if seenAt is null
    -- Idempotent. Never overwrite an existing seenAt (first-seen is the useful value).
```

### U5 · `dismiss` — write

The one that makes "OK / Skip" stick, per user, forever.

```
auth:    required
args:    { entryIds: Id[] }   -- max 20
returns: void
behavior:
    user = requireUser(ctx)
    for id in entryIds:
        verify entry exists AND is published AND matchesAudience
        upsert receipt(user.id, id):
            seenAt      = seenAt ?? now()
            dismissedAt = dismissedAt ?? now()
    -- Idempotent: dismissing an already-dismissed entry is a no-op, not an error.
    -- Never accept a userId argument. The receipt belongs to the caller.
```

### U6 · `recordLinkClick` — write [Optional]

Covers both the footer CTA and any per-media link, so one operation answers "which of the
four screenshots did they actually click?"

```
auth:    required
args:    { entryId, target: "cta" | <mediaItemId> }
behavior:
    user = requireUser(ctx)
    verify the entry is visible to this user
    verify target is "cta" (and the entry has a cta) or matches a media item id on the
      entry — reject unknown targets rather than storing arbitrary client strings
    upsert receipt:
        ctaClickedAt   = ctaClickedAt ?? now()
        clickedTargets = dedupe(clickedTargets + [target])[:10]
        seenAt         = seenAt ?? now()
        dismissedAt    = dismissedAt ?? now()   -- following a link is acknowledgement
```

Dismissing on click is deliberate: the user acted on the announcement, and re-showing it
next session would read as a bug. If the product wants the modal to survive a media-link
click (e.g. the link opens a docs page in a new tab and the user is expected to come back),
make that a documented choice: dismiss on `"cta"`, only mark seen on media links.

### U7 · `getUnreadCount` — read [Optional]

Powers the nav badge.

```
auth: required
returns: number
behavior: same filter as U1 but without the highlight restriction and without the
          MODAL_MAX_ENTRIES cap; count entries published after user.createdAt with no
          dismissal receipt. Cap the count display at "9+" in the UI.
```

---

## Admin operations

Every one of these begins with `requireAdmin(ctx)`. No exceptions.

### A1 · `createDraft`

```
args:    { slug?, title, summary, body?, type, highlight?, version?, cta?, audience? }
returns: Id
behavior:
    admin = requireAdmin(ctx)
    slug = args.slug ?? slugify(title)
    if slug already exists: either reject, or suffix with -2 (pick one and document it)
    insert { ...args, status: "draft", audience: args.audience ?? {mode:"all"},
             highlight: args.highlight ?? (type == "feature"),
             createdBy: admin.id, updatedAt: now() }
```

### A2 · `upsertDraftBySlug` — used by `/changelog:new`

```
args:    same as A1, slug required
returns: Id
behavior: if an entry with this slug exists AND its status is "draft", patch it;
          if it exists and is published, DO NOT silently overwrite — return a conflict
          the CLI can report ("already published; use a new slug or unpublish first").
          Otherwise create. This makes re-running the authoring command safe.
```

### A3 · `updateDraft`

```
args:    { id, patch }
behavior: requireAdmin; reject if entry.status == "archived"; set updatedAt.
          Editing a *published* entry is allowed (typo fixes) but must NOT resurrect it
          for users who already dismissed it — receipts are untouched by edits. Say this
          in the admin UI.
```

### A4 · `publish` — the human gate

```
args:    { id, publishedAt?: timestamp }
returns: { ok: true, audienceSize?: number }
behavior:
    admin = requireAdmin(ctx)
    entry = get(id) or throw
    validate(entry)                      -- the full rule list from 02-data-model.md
    if entry.status == "archived": throw
    patch: status = "published",
           publishedAt = args.publishedAt ?? entry.publishedAt ?? now(),
           publishedBy = admin.id, updatedAt = now()
```

Validation failures must return a **field-level list** the admin form can render inline,
not one opaque string. Missing alt text on media is a validation failure, not a warning.

### A5 · `unpublish`

```
behavior: requireAdmin; status = "draft"; keep publishedAt (so re-publishing preserves
          the original date unless changed). Receipts are NOT deleted — a user who
          dismissed it should not be re-interrupted if it is republished.
```

### A6 · `archive` / `unarchive`

```
archive:   status = "archived", archivedAt = now(). Disappears from every user surface.
unarchive: status = "draft".
```

Prefer archive over delete. **[Optional]** a hard `delete` for admins, which must also
delete the entry's receipts.

### A7 · `listAllEntries`

```
args:    { status?: "draft"|"published"|"archived", cursor?, limit? }
returns: paginated entries with admin projection (includes audience, createdBy,
         publishedBy, and — optional — receipt counts)
behavior: requireAdmin; ordered by updatedAt desc so work-in-progress floats to the top.
```

### A8 · Media operations

Four operations, because media is now an ordered, editable list.

```
generateMediaUploadUrl
    requireAdmin -> short-lived upload target, using the repo's existing storage
    mechanism. Accepts { contentType, sizeBytes } so it can reject early.

addMediaItem
    args:  { entryId, kind, ref, alt, caption?, link?, width?, height?,
             posterRef?, durationMs? }
    returns: { mediaItemId }
    behavior:
        requireAdmin
        reject if entry.media.length >= MEDIA_MAX_ITEMS
        reject empty alt
        validate link.href with the same rules as cta.href
        generate a stable id; order = max(existing order) + 1
        append and patch updatedAt

updateMediaItem
    args: { entryId, mediaItemId, patch: { alt?, caption?, link?|null } }
    behavior: requireAdmin; reject clearing alt to empty; passing link: null removes the
              link (distinguish "not provided" from "explicitly cleared" — a partial patch
              that cannot remove a field is a small trap that bites later).

reorderMedia
    args: { entryId, orderedIds: string[] }
    behavior: requireAdmin; verify the set of ids matches the entry's media exactly
              (no additions, no omissions); rewrite `order` from the array position.

removeMediaItem
    args: { entryId, mediaItemId }
    behavior: requireAdmin; drop the item; renumber remaining `order` to stay contiguous;
              delete the stored file if the repo's storage layer supports it and the ref
              is not shared with another entry.
```

**Validation limits** worth enforcing server-side, since an oversized video in a modal is
a worse client experience than no video at all:

```
images:  <= 5 MB,  formats png/jpg/webp/avif per what the repo already serves
videos:  <= 15 MB, <= 60s, mp4 (h264) or webm
posters: <= 1 MB
```

If the repo has no storage layer, the fallback is: files are committed under the public
assets directory and `addMediaItem` stores that path. Note in the admin UI that this
requires a deploy, and that `removeMediaItem` will not delete the committed file.

### A9 · `previewAudience` [Optional]

```
args: { audience } -> { count: number }
Lets the admin see "this will reach 42 users" before publishing. Skip if counting users
is expensive in this repo.
```

### A10 · `entryStats` [Optional]

```
args: { id } -> { seen: number, dismissed: number, ctaClicked: number }
Uses the by_entry receipt index. This is what tells the developer whether the client
actually looked.
```

---

## Projection

Two shapes. Never return the admin shape to a user-facing surface.

```
publicProjection(entry) = {
  id, slug, title, summary, body, type, version, publishedAt,
  media: entry.media
           .sort(by order)
           .map(m => ({ id: m.id, kind: m.kind, url: <resolved>, alt: m.alt,
                        caption: m.caption, link: m.link,
                        width: m.width, height: m.height,
                        posterUrl: <resolved, video only>, durationMs: m.durationMs }))
           .filter(m => m.url != null),          // drop unresolvable items
  cta,
}
-- Excluded on purpose: status, audience, createdBy, publishedBy, updatedAt, archivedAt.
-- `audience` in particular can name customers or roles.

adminProjection(entry) = full record + resolved media url (+ stats when requested)
```

---

## Security checklist

Walk this before declaring the backend done. Each line is a real bug that has shipped in
implementations of this feature.

- [ ] No user-facing operation accepts a `userId`/`tenantId` argument for identity.
- [ ] Every admin operation calls the repo's real role check, not a client-passed flag.
- [ ] No user-facing read can return a `draft` or `archived` entry — including
      `getEntryBySlug` with a known slug.
- [ ] Audience filtering happens before serialization, on the server.
- [ ] `markSeen`/`dismiss` validate that each id refers to an entry the caller may see,
      so a client cannot create receipts for arbitrary or non-existent ids.
- [ ] Array arguments are length-capped (20) — an unbounded `entryIds` is a cheap DoS.
- [ ] The public projection omits `audience`.
- [ ] Write operations are rate-limited if the repo has a rate limiter; if not, the id
      cap plus idempotency is the mitigation.
- [ ] Markdown `body` is rendered safely on the client (see `04-ui-contracts.md`) —
      admins are trusted, but stored XSS through a changelog is still stored XSS.
- [ ] `cta.href` **and every media item's `link.href`** are validated: internal paths start
      with `/`; external URLs must be `https:` and open with `rel="noopener noreferrer"`.
      Reject `javascript:` and `data:` outright. Per-item links are easy to forget here —
      they are the newer field and they take the same untrusted-ish path.
- [ ] `recordLinkClick` rejects a `target` that is neither `"cta"` nor a real media item id
      on that entry, so a client cannot write arbitrary strings into `clickedTargets`.
- [ ] Media upload accepts only expected content types and enforces the size caps; an
      admin-only upload endpoint is still an upload endpoint.
- [ ] `reorderMedia` verifies the id set matches exactly, so it cannot be used to graft
      another entry's media onto this one.
- [ ] If the repo has RLS/security rules, they are written too — server checks and
      database rules are complementary layers.
