# 05 · Admin console

The surface where a human turns a draft into something clients see. Build this **before**
the user-facing UI: it gives you real data to test with, and it is the surface that makes
the whole feature usable without an agent in the loop.

Route: match the repo's admin convention from discovery (`/admin/changelog`,
`/dashboard/settings/changelog`, `/staff/changelog`). Reuse the repo's existing admin
layout and its guard — do not write a new authorization mechanism.

**Guard on the server too.** A client-side role check that hides the nav link is not
access control; every operation in `03-backend-operations.md` calls `requireAdmin` and
the route itself must be server-guarded the way other admin routes in this repo are.

---

## S1 · Entry list

```
[ Drafts (3) | Published (12) | Archived ]        [+ New entry]

┌───────────────────────────────────────────────────────────────────┐
│ ● draft   Bulk CSV export              feature   all users        │
│           edited 2h ago by Adi                    [Edit] [Publish]│
├───────────────────────────────────────────────────────────────────┤
│ ○ live    Faster search                improve   Acme Corp        │
│           published 11 Aug · 42 seen · 38 dismissed · 9 clicked    │
│                                        [Edit] [Unpublish] [Archive]│
└───────────────────────────────────────────────────────────────────┘
```

- Default tab: **Drafts** — that is where the work is.
- Sort by `updatedAt` desc.
- Status is conveyed by a label, not color alone.
- Engagement numbers only if you implemented `entryStats` (A10). They are the feedback
  loop that tells the developer whether clients actually read what shipped.

## S2 · Editor

One form, mirroring the entry model. Field order below is the order a human thinks in.

| Field | Control | Rules |
|---|---|---|
| Title | text | required, ≤80, live counter |
| Summary | textarea (2 rows) | required, ≤200, live counter, helper: "Plain language. What can the user now do?" |
| Type | segmented control | feature / improvement / fix / announcement |
| Show as popup | toggle | = `highlight`. Default on for feature, off for fix. Helper: "Off means it only appears on the changelog page." |
| Body | markdown textarea + preview tab | optional |
| Media | multi-file drop zone → item list (see S2b) | optional, up to `MEDIA_MAX_ITEMS` (6) |
| Link | label + href pair | optional entry-level CTA; validate href (see below) |
| Version | text | optional |
| Audience | mode selector + picker | default "Everyone" |
| Slug | text, auto-filled from title | editable before first publish; locked after (it is a permalink) |

Behaviors that matter:

- **Autosave the draft** on a debounce, or make Save prominent and warn on navigate-away.
  Losing three paragraphs of release notes to a stray back-button is a bad first impression.
- **Live preview**: render the actual `WhatsNewModal` component with the form's current
  values, in a frame beside the form. This is the single highest-value piece of the
  console — the admin sees exactly what the client will see, including how each screenshot
  crops and whether the title wraps badly. Keep the gallery *interactive* in the preview
  (stepping through items is the point) while the dismiss actions are inert. Add a
  light/dark toggle on the preview frame; screenshots that look great on white regularly
  fall apart on a dark background.
- **Href validation**: if it starts with `/`, check it against the repo's known routes if
  that is cheap (App Router: the `app/` tree; a route manifest if one exists) and warn —
  do not block — when it does not resolve. If it is absolute, require `https:`. Reject
  `javascript:` and `data:` outright.
- **Audience picker** renders only the modes the repo supports: hide tenant targeting in a
  single-tenant app, hide role targeting if the repo has no roles.

## S2b · Media manager

Media is an ordered list of images and videos, each independently captioned and linkable.
The editor needs a small manager, not a single file input.

```
Media                                            3 of 6 · drag to reorder

┌──────────────────────────────────────────────────────────────┐
│ ⠿  [thumb]  export-menu.png · image · 1600×900 · 240 KB      │
│             Alt   The Reports page with the Export menu open │  ← required
│             Cap.  Export lives in the top-right              │  ← optional
│             Link  [Open reports] → /reports                  │  ← optional
│                                                    [Remove]  │
├──────────────────────────────────────────────────────────────┤
│ ⠿  [thumb]  scheduling.mp4 · video · 0:14 · 3.1 MB           │
│             ⚠ No poster frame — one will be generated        │
│             Alt   Choosing a weekly schedule for a report    │
│             Link  [Set up a schedule] → /reports/schedule    │
│                                                    [Remove]  │
└──────────────────────────────────────────────────────────────┘

              [ + Add images or video ]     drop files here
```

Requirements:

- **Multi-select and drag-drop upload**, several files at once, uploading in parallel with
  per-file progress. One failure does not cancel the others.
- **Drag to reorder**, writing through `reorderMedia`. Order is what the client sees; the
  first item is the one digest mode shows, so it should be the strongest.
- **Alt text is required per item.** Block publish with an error naming the item ("Item 3
  of 4 needs alt text"), and mark the offending row. Do not auto-fill alt text from the
  filename — `Screenshot 2026-08-13 at 14.22.11.png` is worse than nothing because it
  looks filled in.
- **Caption is separate from alt**, and the form should say why in one line of helper text:
  the caption is read by everyone, the alt is read by people who cannot see the item.
- **Per-item link** is a label + href pair with the same validation as the entry CTA.
  Helper text: "Where should this screenshot take them?"
- **Client-side pre-checks** before upload: type, size (5 MB image / 15 MB video), and
  video duration. Reject early with a clear message rather than after a 15 MB upload.
- **Poster frames** for video: capture a frame client-side (draw the video to a canvas at
  ~1s) or accept an uploaded poster. Without one, the modal shows an empty box while the
  video loads.
- **Read intrinsic dimensions** on upload and store them — this is what stops the modal
  from jumping as the client steps through the gallery.
- **Removing an item** asks for confirmation only if it is the last one; otherwise just
  remove with an undo toast.
- **Warn, don't block**, when an entry has more than four items or more than one video:
  "Long galleries get skipped. Consider keeping the two clearest." Authors over-collect;
  a nudge at authoring time is cheaper than a client who stops opening the modal.

## S3 · Publish action

The gate. Treat it like a deploy button, because it is one.

```
[Publish]  ->  confirmation dialog:

    Publish "Bulk CSV export"?

    Audience    Everyone (approx. 128 users)         [if previewAudience implemented]
    Popup       Yes — shown once per user on next visit
    Link        /reports?export=1
    Date        Now  ( [ ] backdate to ______ )

    This is visible to customers immediately.
    [Cancel]  [Publish]
```

- Run server-side validation **before** opening the confirmation, and render failures
  inline on the offending fields.
- On success: toast, move the row to Published, show the public permalink.
- Publishing must be idempotent — a double-click cannot produce two publishes or reset
  `publishedAt`.

## S4 · Post-publish controls

- **Unpublish** — back to draft, hidden from users. Receipts survive; a user who already
  dismissed it is not re-interrupted if it is republished. Say that in the confirm dialog.
- **Archive** — permanent removal from all user surfaces, kept for the record.
- **Edit while published** — allowed for typo fixes; changes are live immediately and do
  **not** re-notify anyone. Show that as a note in the editor when status is published.
- **[Optional] Re-notify** — clears dismissal receipts so the modal shows again. Powerful
  and easy to abuse; if you build it, require a typed confirmation and log who did it.

## S5 · Empty and error states

- Empty drafts list: a short prompt plus the New entry button, and a line pointing at the
  CLI flow: "Drafts also arrive from `/changelog:new` in your editor."
- Failed media upload: keep the form state **and the other items**, show the error on the
  failed row only, offer retry on that row without re-entering any text.
- Partial batch upload (4 files dropped, 1 rejected): keep the 3 that worked, name the one
  that failed and why ("scheduling.mov — 24 MB exceeds the 15 MB limit").
- Permission error: if a non-admin somehow reaches the route, render the repo's standard
  forbidden state, not a blank page.

---

## Why draft-then-publish

Worth stating in the code comments, because a future contributor will try to "simplify"
it away:

1. Changelog copy is **client-facing marketing**. An agent that writes and publishes in
   one step can put a typo, a wrong claim, or an internal codename in front of a paying
   customer with nobody in the loop.
2. Publishing is **irreversible in practice**. Unpublishing hides the entry, but anyone
   who already saw the modal saw it.
3. The draft is the **handoff artifact** between the developer's terminal and the
   product decision. The developer describes what shipped; the human decides whether and
   how to say it, and to whom.

The agent's job ends at a well-formed draft and a link to this console.
