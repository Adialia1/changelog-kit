# 04 · User-facing UI contracts

Three components. Each is specified by **states, behavior, and accessibility** — not by
markup. Build them from the host repo's existing primitives (discovery §5). Any pseudo-JSX
here shows structure and logic; the real implementation uses the repo's dialog, button,
badge, and tokens.

The test of success: someone who knows the repo cannot tell these components came from
elsewhere.

---

## C1 · What's New modal

The interruption. Held to a high bar because it costs the user attention.

### Mounting

Mount **once**, in the authenticated app shell found in discovery — the layout that wraps
every signed-in page. Not in individual pages, not in the root layout (which also renders
sign-in screens).

```
<AuthenticatedLayout>
  {children}
  <WhatsNewModal />     // renders null unless there is something to say
</AuthenticatedLayout>
```

### Suppression rules — when it must not appear

Ship these from day one; retrofitting them means angry users first.

1. Auth state is still loading, or the user is signed out.
2. The user is inside a **first-run onboarding** flow (grep the repo for an onboarding
   route/flag and gate on it). A brand-new user must complete onboarding first.
3. The current route is in the **suppression list**: checkout/payment, any full-screen
   editor or wizard, print views, embedded/iframe contexts, error pages. Define the list
   as an exported constant next to the component so it is editable in one place.
4. Another blocking modal is already open. If the repo's dialog primitive does not
   expose that, delay by one tick and check again; never stack dialogs.
5. Viewport is a small mobile screen **and** the repo has a mobile-specific shell that
   makes dialogs awkward — prefer a bottom sheet if the repo has one; otherwise a
   full-screen dialog with the same content.
6. `getPendingHighlights` errored or is still loading. **Failure renders nothing.** No
   error toast, no skeleton, no layout shift. (Invariant #7.)

### States and modes

| Pending entries | Render |
|---|---|
| 0, or loading, or errored | `null` — nothing at all |
| 1 | Single entry card, primary action "Got it" |
| 2–`MODAL_STEPPER_MAX` (3) | **Stepper**: one entry at a time, "Next" until the last, then "Got it", with a step indicator (`2 of 3`) |
| 4–`MODAL_MAX_ENTRIES` (8) | **Digest**: one scrollable modal listing every entry compactly, a single "Got it" that dismisses all |
| more than 8 | Digest of the newest 8 + a line: "and 4 more on the changelog page →" |

Digest mode exists because a stepper past three steps stops being an announcement and
becomes a wizard the user clicks through without reading. When you have shipped six things
at once, a scannable list respects the reader more — they skim, stop at the one that
matters to them, and click its link.

The same entry renders in both modes, at two densities:

- **Stepper (full density)**: title, summary, full media gallery, truncated body, CTA.
- **Digest (compact density)**: type badge, title, summary, **one** media item (the first,
  as a thumbnail strip if there are more), the entry's link, and "Details →" to the
  changelog page. No body markdown.

### Content layout — one entry, full density

Order matters: the user reads top to bottom and most will only read the first two lines.

```
[type badge]  [version chip, if present]
Title                                            (heading, 1–2 lines, no truncation mid-word)
Summary                                          (2 lines, plain language)

┌── media gallery ──────────────────────────────┐
│  [ active item: image or video, 16:9 box ]    │
│  caption text under the item, if any          │
│  [ Try bulk export → ]   <- this item's link  │
│  ● ○ ○ ○      2 / 4      [ ‹ ]  [ › ]         │
│  [thumb][thumb][thumb][thumb]                 │
└───────────────────────────────────────────────┘

[body: rendered markdown, truncated — full text lives on /changelog/{slug}]
────────────────────────────────────────
[Skip (tertiary)]        [entry CTA (primary, if present)]  [Got it / Next]
```

Details that decide whether this feels good:
- **Reserve the media box** from the stored width/height so the dialog does not jump as
  items load. Unknown dimensions → a fixed 16:9 container. This matters more with a
  gallery than with one image: four items of differing aspect ratios will make the dialog
  bounce on every navigation unless the box is fixed.
- The active item is `loading="eager"`; the rest are lazy, but **prefetch the next one**
  so stepping through feels instant.
- Neutral placeholder background from the repo's tokens while loading.
- An item that fails to load is removed from the gallery, not shown broken. If every item
  fails, the gallery disappears and the entry renders as text — still a valid announcement.
- Cap modal height at ~80vh, content area scrolls, page behind never scrolls.
- Long body markdown belongs on the changelog page. Truncate and link out.

### Media gallery — inside one entry

| Items | Render |
|---|---|
| 0 | No gallery. Text-only entry is normal and fine. |
| 1 | The item alone, full width. No dots, no counter, no thumbnails. |
| 2–6 | Active item + dots + counter + prev/next controls. Add a thumbnail strip at 3+. |

**Per-item link.** When `item.link` is present, render it as a visible button or link chip
directly beneath that item — never as an invisible click target on the image alone. An
image that is silently clickable is a link nobody finds. The image *may also* be clickable
as a convenience (with `cursor: pointer` and a hover state), but the chip is what makes it
discoverable.

Clicking a per-item link: `recordLinkClick(entryId, itemId)` → navigate (internal via the
repo's router, external in a new tab with `rel="noopener noreferrer"`) → close the modal.

**Video items.**
- `muted` + `playsInline` always. Never autoplay with sound; a changelog that makes noise
  in an open-plan office is a support ticket.
- Autoplay + loop only when: the item is the active one, `durationMs <= 20s`, and
  `prefers-reduced-motion` is not set. Otherwise show the poster with a play button.
- `controls` for anything longer than 10s, and for every video when reduced motion is on.
- Poster frame always, so the box is filled before the first frame decodes.
- Show a duration chip (`0:12`) so the user knows what they are committing to.
- Pause the video when the user steps to another item or another entry — a playing video
  in a hidden panel wastes battery and confuses screen readers.
- `preload="metadata"`, not `auto` — do not pull 15 MB for an item the user may never see.

**Navigation conflict — read this before wiring keyboard handlers.** There are now two
horizontal navigations on screen: entries (stepper) and media (gallery). Keep them
unambiguous:

- Entry navigation lives **only** in the footer buttons (`Next` / `Got it`). It has no
  keyboard shortcut of its own beyond Tab-to-button-then-Enter.
- Media navigation responds to `ArrowLeft` / `ArrowRight` **only while focus is inside the
  gallery**. Do not attach arrow handlers to the dialog.
- Swipe gestures, if the repo uses them elsewhere, belong to the gallery, not the stepper.
- Never nest two scroll-snap carousels. If that is where the implementation is heading,
  switch the entries to digest mode instead.

### Behavior

```
on mount (auth resolved, not suppressed):
    entries = query getPendingHighlights()
    if entries empty -> render null

on first render of entry i:
    call markSeen([entries[i].id])        // fire-and-forget; failure must not break UI

on "Next":
    advance index, and dismiss the entry being advanced past — so entry i is dismissed
    when the user moves to i+1 or finishes. Entries the user never reached stay pending
    and return next session.

on "Got it" (last entry) / "Skip":
    dismiss(all ids the user actually reached)
    close immediately (optimistic) — do not await the mutation
    if the mutation fails, retry once in the background; a failed dismissal means the
    modal reappears next session, which is an acceptable degradation

on entry CTA click:
    recordLinkClick(entry.id, "cta")   (which also dismisses)
    internal href -> repo's client-side navigation, then close
    external href -> new tab with rel="noopener noreferrer", modal stays closed

on media item link click:
    recordLinkClick(entry.id, item.id)
    same navigation rules; also dismiss the entries reached so far
    -- The user acted on the announcement. Re-showing it next session reads as a bug.

on media navigation (dots / arrows / thumbnails / swipe):
    change active item; pause any playing video; prefetch the neighbour
    -- Media navigation NEVER advances the entry stepper.

in digest mode:
    every entry is visible at once, so mark all of them seen on open
    "Got it" dismisses all listed entries in one call

on Esc / backdrop click:
    same as "Skip" — dismiss what was reached. Do not treat Esc as "ask me again",
    because a user who pressed Esc twice will consider a third appearance a bug.
```

**Do not re-query on every route change.** Query once per session/mount. Cache the
"nothing pending" answer for the session so navigating around the app does not re-fetch.

### Accessibility

Non-negotiable, and the repo's dialog primitive gives most of it for free — which is why
you use it instead of a hand-rolled overlay.

- `role="dialog"` + `aria-modal="true"`, labelled by the title element
  (`aria-labelledby`) and described by the summary (`aria-describedby`).
- Focus moves into the dialog on open, is trapped while open, and returns to the
  previously focused element on close.
- Esc closes. Tab order is: content → CTA → Got it → Skip → close button.
- The close/skip control has an accessible name even if it renders as an icon.
- Every media item has real alt text (required at publish time). In a gallery this is not
  decoration — it is the only way a screen-reader user knows the four items differ.
- **Gallery semantics**: wrap it in a labelled region (`role="group"`,
  `aria-roledescription="carousel"`, `aria-label="Feature screenshots"`). Prev/next
  controls need accessible names ("Previous screenshot"). Dots are buttons with names
  ("Go to item 2 of 4"), not bare divs. The counter is `aria-live="polite"`.
- **Per-item links** get their own accessible name that makes sense out of context: "Try
  bulk export" is good, "Go" is not. If the image is also clickable, it must not create a
  duplicate tab stop — mark the redundant one `aria-hidden` / `tabindex="-1"`.
- **Video**: `<track kind="captions">` when the clip has speech (most product clips do
  not); `controls` for anything over 10s; never autoplay under `prefers-reduced-motion`;
  the play button has a name that says what plays.
- Respect `prefers-reduced-motion`: no slide/scale entrance, no autoplay, no auto-advance
  of gallery items — just instant or opacity-only transitions.
- Step indicator is announced (`aria-live="polite"` on "2 of 3") when it changes.
- Two live regions on screen (entry step + media counter) will talk over each other if
  both change at once. Only the one the user just interacted with should be live.
- Contrast meets the repo's existing standard in **both** themes — check dark mode.

### Markdown safety

The `body` is authored by an admin, but render it through the repo's existing markdown
renderer with its sanitizer configured. If the repo has none, either render plain text
with paragraph breaks only (safe and usually sufficient) or add the repo's preferred
sanitizing renderer. Never inject raw HTML strings into the DOM — an admin account is
trusted, but stored XSS through a changelog is still stored XSS.

---

## C2 · Changelog page

Route: follow the repo's convention. Default `/changelog` inside the authenticated
section; `/changelog/[slug]` (or `#slug` anchor) for deep links.

### Layout

```
Header:  "What's new"  +  short subtitle  +  [filter by type, optional]
Body:    entries grouped by month (or by `version` when most entries carry one)

  ── August 2026 ─────────────────────────
  [badge feature]  Bulk CSV export        [v2.4]     13 Aug
  Summary line.
  [media: same gallery component as the modal, or a thumbnail row that opens a lightbox]
  [body markdown]
  [CTA link →]

Footer:  pagination or infinite scroll
```

Media on the page: reuse the gallery component from the modal rather than writing a second
one. Two differences are worth it — all items may be `loading="lazy"` here (they are below
the fold), and videos never autoplay on a list page, where several would compete. Per-item
links behave exactly as in the modal, minus the "close the modal" step.

### Behavior

- Newest first, by `publishedAt`.
- Paginate with `listEntries`; infinite scroll only if the repo already does that
  elsewhere, otherwise a "Load more" button (simpler, more accessible, back-button-safe).
- On mount, `markSeen` the visible entries so the unread badge clears. Do **not** call
  `dismiss` here — a user browsing history should still get the modal for something they
  have not acknowledged. (Alternative, if the user prefers: dismissing on page view is
  defensible; make it a documented decision in the Adaptation Map, not an accident.)
- Deep link: scroll to and highlight the targeted entry; if the slug is unknown or not
  visible to this user, render the list with a quiet "That entry isn't available" notice
  rather than a 404 shell.
- Empty state: a real sentence ("Nothing shipped yet — check back soon"), styled like the
  repo's other empty states.
- Loading: the repo's skeleton component, matching the entry card shape.

### Where the link lives

Add an entry point in the existing navigation — usually the user/help menu, footer, or a
settings section. Match the repo's icon set. Do not invent a new nav region.

---

## C3 · Unread badge [Optional but recommended]

A small dot or count next to the changelog nav link.

- Data: `getUnreadCount`, polled at most once per session (or via the repo's realtime
  layer if it has one — Convex queries are reactive by default, so this is free there).
- Renders nothing at 0. Caps display at `9+`.
- `aria-label="3 unread updates"`; never convey the state by color alone (a bare colored
  dot needs an accessible label).
- Clears when the changelog page is viewed (via `markSeen`).

---

## Design adaptation rules

The generic-looking version of this feature is the failure mode. Rules that prevent it:

1. **Reuse, never re-create.** Dialog, Button, Badge, Card, ScrollArea, Skeleton — all
   come from the repo. If the repo has `components/ui/dialog.tsx`, the modal is built on
   it, period.
2. **Tokens only.** No hex codes, no arbitrary pixel values that bypass the repo's spacing
   scale. Type badges map to the repo's existing semantic colors (success/info/warning),
   not new ones.
3. **Match the file layout.** If components live in `src/components/<domain>/`, changelog
   components go in `src/components/changelog/`. Match the export style (default vs named),
   the "use client" convention, and the prop-typing style.
4. **Match the data-fetching idiom.** If the repo uses server components + props, do that.
   If it uses a `useQuery` hook everywhere, do that. Do not introduce a second pattern.
5. **Copy the closest existing component's structure.** Before writing the modal, read the
   repo's most similar existing dialog and mirror its shape — prop naming, class ordering,
   how it handles loading, how it composes the footer.
6. **RTL and i18n** (if discovery found them): logical properties, mirrored chevrons, all
   static strings through the i18n system.
7. **Both themes.** Render-check light and dark.
8. **No new dependencies.** If a carousel is needed and the repo has none, a
   state-index-plus-conditional-render is a carousel. Do not add a library for it.

---

## Pseudo-implementation

Structure and logic only — rewrite in the repo's idiom.

```jsx
function WhatsNewModal() {
  const auth = useAuthState();                       // repo's hook
  const route = useCurrentRoute();
  const suppressed = SUPPRESSED_ROUTES.some(p => route.startsWith(p)) || isOnboarding(auth);

  const skip = !auth.isAuthenticated || auth.isLoading || suppressed;
  const entries = useChangelogPending({ enabled: !skip });   // U1; returns [] on error

  const [index, setIndex] = useState(0);
  const [open, setOpen] = useState(false);
  const reached = useRef(new Set());

  useEffect(() => {
    if (!skip && entries?.length) setOpen(true);
  }, [skip, entries]);

  useEffect(() => {
    const current = entries?.[index];
    if (open && current && !reached.current.has(current.id)) {
      reached.current.add(current.id);
      markSeen([current.id]);                        // fire-and-forget
    }
  }, [open, index, entries]);

  if (skip || !entries?.length) return null;

  const entry = entries[index];
  const isLast = index === entries.length - 1;

  const close = () => {
    setOpen(false);
    dismiss([...reached.current]);                   // optimistic; retry once on failure
  };

  return (
    <Dialog open={open} onOpenChange={(o) => !o && close()}>   {/* repo's Dialog */}
      <DialogContent aria-labelledby="cl-title" aria-describedby="cl-summary">
        <Badge variant={badgeVariantFor(entry.type)}>{t(entry.type)}</Badge>
        <DialogTitle id="cl-title">{entry.title}</DialogTitle>
        <DialogDescription id="cl-summary">{entry.summary}</DialogDescription>

        <MediaGallery
          items={entry.media}
          onLinkClick={(item) => {
            recordLinkClick(entry.id, item.id);
            openLink(item.link.href);
            close();
          }}
        />

        {entry.body && <Markdown className="prose-per-repo">{truncate(entry.body)}</Markdown>}

        <DialogFooter>
          {entries.length > 1 && <span aria-live="polite">{index + 1} / {entries.length}</span>}
          <Button variant="ghost" onClick={close}>{t("skip")}</Button>
          {entry.cta && (
            <Button onClick={() => {
              recordLinkClick(entry.id, "cta"); openLink(entry.cta.href); close();
            }}>
              {entry.cta.label}
            </Button>
          )}
          <Button variant={entry.cta ? "outline" : "default"}
                  onClick={() => isLast ? close() : setIndex(i => i + 1)}>
            {isLast ? t("gotIt") : t("next")}
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}
```

The gallery, kept deliberately dependency-free — an index in state *is* a carousel:

```jsx
function MediaGallery({ items, onLinkClick }) {
  const [active, setActive] = useState(0);
  const reducedMotion = usePrefersReducedMotion();
  if (!items?.length) return null;

  const item = items[active];
  const ratio = item.width && item.height ? item.width / item.height : 16 / 9;

  return (
    <div role="group" aria-roledescription="carousel" aria-label={t("featureMedia")}>
      <MediaFrame ratio={ratio}>                     {/* fixed box: no layout jump */}
        {item.kind === "image" ? (
          <img src={item.url} alt={item.alt} loading={active === 0 ? "eager" : "lazy"} />
        ) : (
          <video
            key={item.id}                            /* remount on change = stops playback */
            src={item.url}
            poster={item.posterUrl}
            aria-label={item.alt}
            muted playsInline
            preload="metadata"
            autoPlay={!reducedMotion && item.durationMs <= 20000}
            loop={!reducedMotion && item.durationMs <= 20000}
            controls={reducedMotion || item.durationMs > 10000}
          />
        )}
      </MediaFrame>

      {item.caption && <p className="text-muted">{item.caption}</p>}

      {item.link && (
        <Button size="sm" variant="secondary" onClick={() => onLinkClick(item)}>
          {item.link.label}
        </Button>
      )}

      {items.length > 1 && (
        <nav onKeyDown={(e) => {                     /* arrows only inside the gallery */
          if (e.key === "ArrowRight") setActive(i => Math.min(i + 1, items.length - 1));
          if (e.key === "ArrowLeft")  setActive(i => Math.max(i - 1, 0));
        }}>
          <IconButton aria-label={t("previousItem")} disabled={active === 0}
                      onClick={() => setActive(i => i - 1)} />
          <span aria-live="polite">{active + 1} / {items.length}</span>
          <IconButton aria-label={t("nextItem")} disabled={active === items.length - 1}
                      onClick={() => setActive(i => i + 1)} />

          {items.map((it, i) => (
            <button key={it.id} aria-label={t("goToItem", { n: i + 1, total: items.length })}
                    aria-current={i === active} onClick={() => setActive(i)}>
              <img src={it.posterUrl ?? it.url} alt="" aria-hidden="true" />
            </button>
          ))}
        </nav>
      )}
    </div>
  );
}
```

Note `key={item.id}` on the video: remounting on change is the simplest correct way to stop
the previous clip. Without it, React reuses the element, the old video keeps playing, and
you get audio-free but very confusing behavior on slow connections.
