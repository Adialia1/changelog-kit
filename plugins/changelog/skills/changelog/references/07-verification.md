# 07 · Verification

This is not a static checklist to tick off. It is a **procedure for deriving tests from
the install you just performed**, running them against the real system, and reporting
honestly. Run it after every install and after any change to changelog code.

Rule: a check you did not actually execute is reported as **not run**, never as passed.
A partial pass with an honest note is worth more than a green report that is wrong.

---

## Phase 0 · Derive the test matrix

Read `changelog/ADAPTATION-MAP.md` and turn it into concrete commands. You cannot know in
advance whether this repo runs `npm test`, `pnpm vitest`, or `bin/rails test`. Fill in:

```
TYPECHECK   = <repo's typecheck command, or "none">
LINT        = <repo's lint command, or "none">
TEST        = <repo's test command, or "none">
RUN_APP     = <how to start the app locally>
BACKEND_CLI = <how to invoke a backend function from the terminal, dev environment>
SEED_USERS  = <how to create/obtain two distinct test users, A and B>
ADMIN_USER  = <how to obtain a user with the publishing role>
```

If `SEED_USERS` has no answer, solve it before continuing — the entire per-user isolation
guarantee is untestable without two users. Options: an existing seed script, a signup
flow you can drive twice, or two rows inserted directly in dev.

---

## Phase 1 · Static checks

Cheap, run first.

1. **Build integrity**: run `TYPECHECK`, `LINT`, `TEST`. Zero new errors attributable to
   changelog files. Pre-existing failures elsewhere: note them, do not fix them unasked.
2. **Schema present**: the entry and receipt tables exist in the schema file, with every
   index from `02-data-model.md`.
3. **Read every file you wrote**, looking specifically for:
   - a user-facing function that takes a `userId` argument → invariant #1 violation
   - a user-facing read that lacks a `status == "published"` condition → invariant #2
   - an admin function missing `requireAdmin` → grep every exported admin symbol
   - audience filtering done in a component instead of on the server → invariant #3
4. **Dead-link check**: every `cta.href` in seeded entries resolves to a real route.
5. **Convention diff**: open the nearest sibling files (an existing component, an existing
   backend module) and compare side by side. Do the imports, naming, error handling, and
   file structure match? If not, fix before proceeding — this is the difference between
   "installed" and "integrated".

---

## Phase 2 · Backend behavior

Run these against a **dev/local environment**. Never against production. Use `BACKEND_CLI`
or a test file, whichever the repo supports better.

Each case is written as: setup → action → expected. Report each individually.

| # | Case | Expected |
|---|---|---|
| B1 | Create a draft; call the user-facing list and modal queries as user A | Draft appears in **neither**. Invariant #2. |
| B2 | Fetch the draft's slug via `getEntryBySlug` as user A | `null` — not a 403, not the entry |
| B3 | Publish it (as admin); repeat B1 as user A | Entry now appears in both |
| B4 | Call the publish operation as a **non-admin** user | Rejected with a permission error |
| B5 | Dismiss as user A; re-run the modal query as A | Entry absent |
| B6 | Re-run the modal query as **user B** | Entry still present. **This is the isolation guarantee — the single most important test in this suite.** |
| B7 | Dismiss twice as user A | Second call succeeds, no duplicate receipt, `dismissedAt` unchanged |
| B8 | Create user C with a creation time **after** the publish; run the modal query as C | Entry absent. Invariant #5 (no backfill blast). |
| B9 | Publish an entry with `highlight = false`; modal query vs list query | Absent from modal, present in the list |
| B10 | Publish with `audience = { mode: "users", userIds: [A] }`; query as A, then as B | Visible to A only; B's payload contains no trace of it |
| B11 | Call `dismiss` with a random/non-existent id | Rejected or ignored; no receipt row created |
| B12 | Call `dismiss` with 500 ids | Rejected by the length cap |
| B13 | Publish an entry with 4 media items where item 3 has empty alt | Rejected at publish, with an error naming **item 3** — not a generic "media invalid" |
| B14 | Publish an entry with a duplicate slug | Rejected or suffixed per the documented rule — whichever the install chose |
| B15 | Unpublish an entry user A dismissed, then republish | A is **not** re-shown; receipts survived |
| B16 | Archive a published entry; re-run both user queries | Absent from both |
| B17 | Publish 12 highlighted entries; run the modal query | Returns at most `MODAL_MAX_ENTRIES` (8), newest first; 4 or more entries means digest mode |
| B18 | Publish an entry dated older than `MODAL_LOOKBACK_DAYS`; modal query | Absent from modal, present on the page |
| B19 | Call any user-facing operation while signed out | Returns empty / unauthorized per spec — never a server crash |
| B20 | Inspect the raw payload returned to a user client | Contains no `audience`, `status`, or internal ids beyond what the public projection allows |

### Media and links (B21–B30)

| # | Case | Expected |
|---|---|---|
| B21 | Publish an entry with 4 media items added out of order, then `reorderMedia` | The user-facing query returns them sorted by `order`, matching the admin's arrangement |
| B22 | Inspect a returned media array | Every item has a resolved `url` — no storage ids, no nulls, poster urls resolved for videos |
| B23 | Delete the underlying file of item 2, then query | Item 2 is dropped from the array; the entry and its other 3 items still render |
| B24 | Add a 7th media item | Rejected by `MEDIA_MAX_ITEMS` |
| B25 | `recordLinkClick(entryId, "cta")` and `recordLinkClick(entryId, <item id>)` | Both stored; `clickedTargets` contains both, deduped; `ctaClickedAt` set once, on the first |
| B26 | `recordLinkClick` with a media id belonging to a **different** entry | Rejected — a client cannot write arbitrary targets |
| B27 | `recordLinkClick` on any target | The entry is also dismissed for that user (or marked seen only, if the install documented that variant — check the Adaptation Map, then test what it says) |
| B28 | Publish an entry whose media item link is `javascript:alert(1)` | Rejected at publish |
| B29 | `reorderMedia` with an id list that omits one item, or adds a foreign id | Rejected; existing order unchanged |
| B30 | Upload a 40 MB video / a `.exe` renamed to `.mp4` | Rejected by size and content-type checks |

Where the repo has a backend test framework, **write B1–B10 plus B21, B25, B26 and B28 as
committed tests**. They are the regression net for the invariants and the media contract,
and they are cheap. The rest may be run manually and reported.

---

## Phase 3 · UI behavior

Run the app. Drive it — with browser automation if available, by hand-instructing the
user if not (say so explicitly in the report).

1. **First appearance**: sign in as user A with one published highlight pending → the
   modal appears, over the authenticated shell, not on the login screen.
2. **Content correctness**: title, summary, badge, media, and CTA label match the entry.
   No image is stretched, cropped wrong, or missing alt.
3. **Dismiss and reload**: click "Got it", reload the page → modal does **not** return.
4. **Cross-session**: sign out, sign back in as A → still gone.
5. **Cross-user**: sign in as user B → modal appears for B. Dismiss as B; A stays
   dismissed. (The UI-level mirror of B6.)
6. **CTA**: click it → lands on the correct route, modal closes, does not return.
7. **Esc / backdrop**: reopen with a fresh entry → Esc dismisses and it does not return.
8. **Multi-entry stepper**: with 2–3 pending → "Next" advances, the step indicator updates,
   the last step shows "Got it", all reached entries are dismissed.
8b. **Digest mode**: with 5 pending → one scrollable modal listing all five, and a single
   "Got it" dismisses every one (verify by reloading — none return).
8c. **Gallery**: an entry with 4 items → dots, counter, prev/next and thumbnails all
   navigate; the counter tracks; the dialog does **not** change height or jump as items
   change; the entry stepper does **not** advance when the gallery does.
8d. **Per-item links**: each item's link is visible without hovering, navigates to that
   item's own href, closes the modal, and does not return on reload. Then check the
   database: the correct **item id** landed in `clickedTargets`. A gallery where every
   link records `"cta"` looks perfect on screen and is worthless as data.
8e. **Video**: autoplays muted and loops for a short clip; shows controls for a long one;
   pauses when you move to another item or another entry; the poster fills the box before
   load; nothing ever plays audio.
8f. **Empty gallery**: an entry with no media renders as a clean text card — no empty box,
   no stray counter, no leftover padding.
9. **Suppression**: navigate directly to a suppressed route (checkout, onboarding) with a
   pending entry → no modal.
10. **Failure tolerance**: break the query deliberately (stop the backend, or point it at
    a bad id) → the app renders normally, no error toast, no blank screen. Restore after.
11. **Changelog page**: lists published entries newest-first; pagination works; deep link
    to a slug scrolls/highlights; empty state renders when nothing is published.
12. **Badge**: shows a count when unread; clears after viewing the page.
13. **Themes**: light and dark, both legible, tokens respected.
14. **Responsive**: ~375px wide and desktop. Nothing clipped, no horizontal scroll, the
    primary action reachable without scrolling on mobile.
15. **RTL**, if applicable: layout mirrors, chevrons flip, text aligns correctly.
16. **Keyboard only**: Tab into the modal, reach every control including gallery dots,
    prev/next, thumbnails and per-item links; arrows move the gallery only while focus is
    inside it and never advance the entry stepper; Esc closes; focus returns to where it
    was. With a screen reader if you have one: the four items announce as four distinct
    things, not "image, image, image, image".
17. **Reduced motion**: with the OS setting on, no slide/scale animation, no video
    autoplay, no auto-advancing gallery.
18. **Slow network** (throttle to 3G): the media box is reserved and grey rather than
    collapsed; the dialog does not reflow when items arrive; the video shows its poster.

Capture screenshots of the modal (light + dark) and the changelog page. They go in the
report and double as the proof the install works.

---

## Phase 4 · Admin console

1. Non-admin user navigating to the admin route → forbidden, both in the UI and if the
   underlying operation is called directly.
2. Create a draft through the form → appears in the Drafts tab.
3. Live preview matches what the modal actually renders (compare against the Phase 3
   screenshot).
4. Publish with a missing required field → inline field-level errors, nothing published.
5. Publish valid → confirmation dialog states audience and popup behavior; on confirm,
   the entry goes live and a user client picks it up.
6. Media upload → drop 3 files at once; all three upload with individual progress, land in
   storage, and their URLs resolve in the preview.
7. Media validation → an oversized video and a wrong content type are rejected with clear
   per-row messages, and the valid files in the same batch still succeed.
8. Reorder by drag → the new order persists after reload and matches what the modal shows.
9. Per-item fields → alt required (publish blocked, offending row marked), caption and
   link save correctly, and clearing a link actually removes it.
10. Unpublish → disappears from user surfaces within one refresh.
11. Double-click Publish → single publish, `publishedAt` unchanged.

---

## Phase 5 · Report

Use `assets/install-report.template.md`. Structure:

```
## Changelog install — verification report

Environment: <dev deployment / local>
Commands:    TYPECHECK ✓  LINT ✓  TEST ✓ (3 new tests)

Backend      27/30 passed · 3 not run
  ✓ B1–B10, B13–B17, B19–B29
  ⚠ B11  dismiss with a bogus id returns success instead of rejecting — FIXED, retested
  ○ B12  not run: no way to construct a 500-id array through the available CLI
  ○ B18  not run: requires a backdated publish; needs admin backdating UI (built, untested)
  ○ B30  not run: no oversized sample video on hand

UI           21/23 passed
  ✓ ...
  ✗ B-UI-14  modal footer buttons wrap awkwardly at 375px — FIXED (stacked below 400px)
  ○ B-UI-15  RTL not applicable: single-locale app

Admin        11/11 passed

Screenshots: <paths>

Known gaps / follow-ups
  - No rate limiting on dismiss; repo has no rate limiter. Mitigated by the 20-id cap.
  - Audience preview count not implemented (counting users is expensive here).

Files added / modified
  <list>
```

Rules for the report:
- Every ✗ is either fixed and retested, or listed as an open gap with a reason.
- Every ○ says **why** it was not run.
- Nothing is marked ✓ unless you observed the result yourself.

---

## Continuous checks

- Re-run Phases 1 and 2 after any change to changelog backend code.
- The B1–B10 committed tests should run in CI with the rest of the suite.
- After the **first real publish** in production, verify once as a real user account that
  the modal appears and dismisses — then never test in production again.
