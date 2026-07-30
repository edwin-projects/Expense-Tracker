# Project Instructions

Single-file app: everything lives in `index.html` (inline CSS + HTML + JS). No build step.
Data syncs across devices via Supabase REST, mirrored to localStorage. Deployed as a static
page (GitHub Pages).

## Git Workflow
- Always push final changes to the `main` branch unless explicitly told otherwise.
- Merge feature branches into `main` and push after completing any task.

## Cache-busting — bump the version on EVERY deploy
GitHub Pages puts a ~10-min HTTP cache on `index.html`, so without this a change lags on
devices that already have the app open or installed. On each change that ships, bump
`const APP_VERSION` in `index.html` **and** write the identical value into `version.json`
(format `YYYY-MM-DD-NN`).

The app fetches `version.json` with `cache:'no-store'` on load, on every foreground and every
2 minutes. When the live version differs from the baked-in one it logs whoever is signed in
out and reloads, so the next login runs the new build — the PIN screen is the signal that
something changed. A network-first service worker (`sw.js`) is the second layer. Do not remove
either.

Keep the two values identical. If `version.json` runs ahead of the deployed `APP_VERSION`,
clients try to update on every check (a per-session guard stops the reload loop, but it's
still wrong). An update is deferred while a modal is open, an amount/description is half-typed,
or writes are still queued for Supabase — never interrupt an entry in progress.

## Deletion is never destructive
Every delete moves the record into the Recently Deleted bin (`S.trash`, synced via the optional
`deleted_items` table) instead of dropping it. Nothing auto-purges. Keep it that way: if you add
a new record type, give it the same treatment via `trashRecord()` / `restoreFromTrash()`.

## Design Principles
- Aesthetic: premium and minimal, not flowery or decorative. Favour elegance and restraint.
- Numbers: use the clean sans-serif ('Outfit'), never a decorative serif. Apply
  `font-variant-numeric: tabular-nums` so digits stay the same width and align
  cleanly in columns/rows — figures must not visually "dance" up and down.
- The ₹ symbol and all numeric values follow the same standard sans-serif styling.
