---
# changelog/entries/YYYY-MM-DD-<slug>.md
# Git-tracked source of one changelog entry. Synced to the database as a DRAFT,
# keyed on `slug` — re-running the sync updates the draft instead of duplicating it.
# A human publishes from the admin console. This file never publishes anything.

slug: bulk-csv-export           # required · lowercase-kebab · permanent (it is a permalink)
title: Bulk CSV export          # required · <= 80 chars · no trailing period
type: feature                   # required · feature | improvement | fix | announcement
highlight: true                 # true = eligible for the What's New modal
                                # false = changelog page only

summary: >                      # required · <= 200 chars · plain language, benefit first
  Export any report to CSV, including filtered views.
  Files up to 50,000 rows.

version: "2.4"                  # optional · release label shown as a chip

media:                          # optional · ordered gallery, up to 6 items
                                # omit the whole block for a text-only entry
                                # order here IS the order the client sees

  - file: ./2026-08-13-bulk-csv-export-1.png   # relative to this file
    alt: The Reports page with the Export menu open, showing CSV and Excel options.
    caption: Export lives in the top-right of every report.   # optional, <= 120 chars
    link:                       # optional · per-item destination
      label: Open reports       # <= 24 chars · verb first
      href: /reports
    width: 1600                 # intrinsic pixels · lets the modal reserve space
    height: 900

  - file: ./2026-08-13-bulk-csv-export-2.mp4
    kind: video                 # inferred from the extension if omitted
    alt: Choosing a weekly schedule so the export arrives by email every Monday.
    poster: ./2026-08-13-bulk-csv-export-2-poster.png   # strongly recommended
    durationMs: 14000           # keep clips under 20s · always muted
    link:
      label: Set up a schedule
      href: /reports/schedule
    width: 1280
    height: 720

cta:                            # optional · the entry's primary action, in the footer
  label: Try bulk export        # <= 24 chars · verb first
  href: /reports?export=1       # internal path preferred; external must be https
                                # independent of per-item links: an entry can have
                                # linked media and no CTA, or a CTA and no media

audience:                       # optional · defaults to everyone
  mode: all                     # all | roles | tenants | users
  # roles: [admin, manager]
  # tenantIds: [tenant_abc]
  # userIds: [user_123]
---

<!--
Optional markdown body. Shown in full on the changelog page; truncated in the modal with
a "Read more" link. Omit entirely for small changes — a good summary is often enough.

Write for the customer, not for the team. No PR numbers, ticket ids, or component names.

Media notes:
  - Every item needs `alt`. In a gallery it is the only thing telling a screen-reader user
    that item 2 differs from item 3. Describe what the item shows, not "screenshot".
  - `caption` is read by everyone; `alt` is read by people who cannot see the item.
  - Give an item a `link` when that specific screenshot has its own destination. If it
    would point at the same place as the entry `cta`, leave it off.
  - Two or three items is the sweet spot. Six is the cap, not a target.
-->

You can now export any report straight to CSV — including the view you have filtered on
screen, so what you download is exactly what you see.

**How it works**

1. Open any report.
2. Click **Export** in the top right.
3. Choose CSV. Larger exports arrive by email when they are ready.

Exports are capped at 50,000 rows per file. For anything larger, split by date range.
