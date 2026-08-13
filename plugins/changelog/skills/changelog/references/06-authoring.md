# 06 · Authoring an entry

Used by `/changelog:new`, and by any session where the user says "add a changelog entry
for what we just shipped."

Output of this phase: a **draft**, in the target environment, plus an entry file committed
to the repo, plus a link to the admin console. Never a published entry.

---

## Step 1 · Work out what shipped (before asking)

Do the homework first, then ask a short, informed question. Nobody wants to be interviewed
about work you could have read.

```
git log --oneline -20
git diff --stat <last-entry-commit-or-tag>..HEAD
ls changelog/entries/ | tail -3        # what was already announced, and in what voice
```

Signals worth extracting:
- New routes/pages added → almost certainly user-visible, and gives you the CTA href.
- New user-facing components or form fields → a feature.
- Changes confined to config, tests, CI, refactors, deps → **not changelog-worthy**. Say
  so rather than manufacturing an announcement.
- The conversation itself: if this session just built the thing, you already know it
  better than git does. Use that.

Then propose, don't interrogate:

> Looks like you shipped bulk CSV export on the reports page (`/reports`, new export
> dialog, 6 commits). Draft entry:
>
> **Bulk CSV export** — *feature*
> "Export any report to CSV, including filtered views. Up to 50,000 rows per file."
> Link: "Try it" → /reports?export=1 · Popup: yes
>
> Want a screenshot on this one, and who should see it?

## Step 2 · Interview — only what you cannot infer

Ask in one batch, not one at a time. Four things at most:

1. **Media** — propose the shot list first ("I'd capture three: the Export button, the
   dialog, the result — each linking to its own page"), then let them cut, add, or hand
   you files. Never ask the open question "do you want a screenshot?" when you can propose
   the gallery and have them edit it.
2. **Audience** — everyone, or a specific role/tenant/user. Skip this question entirely
   in a single-tenant app with no roles.
3. **Popup or page-only** — default from type; confirm only for borderline cases.
4. **Anything the client should know that the code doesn't say** — limits, caveats,
   "requires the Pro plan", "rolling out over the week".

### When several things shipped at once

Common after a sprint. **One entry per user-visible change** — do not merge four features
into one entry with eight screenshots. The modal handles multiple entries natively (a
stepper up to 3, a scannable digest beyond that), and separate entries mean each gets its
own title, its own link, and its own read state.

So: list the user-visible changes, confirm the list with the user, then run steps 3–6 per
change. Set `highlight: false` on the minor ones — three popups and five page-only entries
is a much better client experience than eight popups.

## Step 3 · Media

An entry holds an **ordered gallery** of up to 6 images and videos, and **each item can
carry its own link**. Use that: one item per step of the flow, each pointing where it
belongs, beats one screenshot of everything.

A useful default for a feature with a multi-step flow:

```
1. the entry point         "the new Export button on Reports"     link → /reports
2. the thing it opens      "the export dialog with format options" link → /reports?export=1
3. the result              "an exported CSV opened in a spreadsheet"  (no link)
```

Two or three items is the sweet spot. Six is the cap, not a target — long galleries get
skipped, and a skipped gallery is worse than one good screenshot.

**Acquiring the items**, in order of preference:

1. **Capture them.** If the app runs locally and you have browser automation, drive the
   actual flow and shoot each step. Prefer a focused crop of the feature over a full-page
   shot with an empty sidebar. Use realistic seed data — a screenshot of an empty state
   sells nothing. For a flow worth showing in motion, record a short clip instead of three
   stills.
2. **Take paths.** The user drops files; copy them next to the entry file, in order.
3. **Skip.** A good summary with no media beats bad media.

**Video specifics** — only when motion carries information a still cannot (a drag
interaction, a live-updating chart, a multi-step animation):
- Keep it under 20 seconds. Under 10 is better.
- No audio. It plays muted and the client may never hear it.
- Start on the interesting frame; trim the mouse wandering to the button.
- mp4 (h264) or webm, under 15 MB. Convert a GIF to mp4 if you have the tooling —
  typically 10× smaller for the same clip.
- Supply a poster frame, or the modal shows an empty box while it loads.

**For every item**, before writing the entry:
- Resize images to ≤1600px wide and compress if the repo has tooling.
- Record intrinsic width/height (and duration for video) — this is what stops the modal
  from jumping as the client steps through.
- **Write the alt text yourself**, per item, and show all of them to the user for
  correction in one batch. Describe what *that item* demonstrates, not "screenshot of the
  app". In a gallery the alt text is the only thing distinguishing item 2 from item 3 for
  a screen-reader user.
- Decide the **per-item link**. Ask yourself: if the client is looking at this exact
  screenshot and wants to try it, where should they land? If the answer is the same page
  as the entry-level CTA, leave the item link off rather than duplicating the button.
- Order them as the client would experience the flow, not the order you captured them.
- Upload through the operations the install created (A8), or copy to the public assets
  path if that was the chosen storage fallback.

## Step 4 · Write the entry file

Path: `changelog/entries/YYYY-MM-DD-<slug>.md`, media alongside as
`changelog/entries/YYYY-MM-DD-<slug>.png`.

Use `assets/entry.template.md`. The file is the reviewable, git-tracked source of the
copy; the database row is what production reads. They are kept in sync by the upsert in
Step 5 (keyed on slug), which makes re-running the command safe.

## Step 5 · Create the draft in the target environment

Use the mechanism the install recorded in `changelog/ADAPTATION-MAP.md` — typically a
script the install created, wrapping the repo's own CLI:

```
# examples; the real command is whatever the Adaptation Map says
npx convex run changelog/admin:upsertDraftBySlug --prod '<json>'
node scripts/changelog-sync.mjs changelog/entries/2026-08-13-bulk-csv-export.md --env prod
```

Rules:
- **Confirm the environment out loud before writing to production.** "Creating a draft in
  *production* (deployment `acme-prod`) — ok?" Draft or not, it is the customer's database.
- If the slug already exists as a published entry, do not overwrite it. Report the
  conflict and offer a new slug.
- If the push fails (no credentials, wrong env), leave the entry file in place and tell
  the user exactly which command to run themselves.

## Step 6 · Hand off

```
Draft created: "Bulk CSV export"
  File   changelog/entries/2026-08-13-bulk-csv-export.md
  Env    production (draft — not visible to anyone yet)
  Review https://app.example.com/admin/changelog

Publish it there when the copy looks right.
```

Do not commit unless the user asked you to commit, and never push.

---

## Voice: write for the client, not the changelog

This is the part that determines whether the feature is worth anything. The person
reading is a customer who is "feeling lost about what you shipped" — not a developer
reading release notes.

**Rules**

1. **Lead with what they can now do.** "Export any report to CSV" — not "Added CSV
   export endpoint".
2. **Second person, present tense.** "You can now…", "Reports now include…".
3. **No internal vocabulary.** No PR numbers, ticket ids, component names, table names,
   library names, or refactor talk. If the client would have to ask "what's a webhook
   retrier?", rewrite it.
4. **Summary ≤ 2 sentences.** The first sentence carries the benefit. The second, at
   most, carries a limit or a caveat.
5. **Concrete numbers beat adjectives.** "Loads in about 2 seconds instead of 15" over
   "significantly faster".
6. **Name the surface.** "on the Reports page" tells them where to look, and pairs with
   the CTA.
7. **Fixes get honesty without drama.** "Fixed: invoices over $10,000 displayed with the
   wrong currency symbol." No "improved stability".
8. **No hype.** No "excited to announce", "game-changing", "revolutionary". Clients
   translate hype into distrust.
9. **Match the product's existing voice.** Read the repo's marketing copy or previous
   entries first. A formal financial product and a playful consumer app do not sound
   alike.
10. **One entry per user-visible change.** Do not bundle five unrelated things into
    "Various improvements" — that is exactly the opacity the client complained about.

**Before / after**

| Don't | Do |
|---|---|
| "Refactored the matching engine and added a new scoring pipeline (#442)" | "Matches are now ranked by how well they fit your criteria, so the best options appear first." |
| "Fixed bug in date handling" | "Fixed: bookings made after 11pm sometimes showed the wrong date." |
| "Added new dashboard widgets" | "Your dashboard now shows this month's revenue and open invoices at a glance." |
| "Improved performance" | "Reports with more than 1,000 rows now load in about 2 seconds instead of 15." |
| "Integrated Twilio for notifications" | "You'll get a text message when a client replies — turn it on in Settings → Notifications." |

**Title**: ≤ 8 words, no trailing period, names the thing.
**CTA label**: verb-first, ≤ 24 chars — "Try bulk export", "Open settings", "See the report".
