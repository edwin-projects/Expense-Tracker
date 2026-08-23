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

## Numbers are Indian — always
Grouping is Indian (`toLocaleString('en-IN')` → `12,34,567`), and compact amounts use the
Indian scale: **thousand → lakh → crore**. A figure of a lakh or more is never scaled in
thousands: `₹1.37L`, never `₹137k`. Every compact display goes through `fmtShort()`; don't
hand-roll `/1000` in a template. Full figures are whole rupees (`Math.round`) — paise are
noise at these sizes; the one exception is the SMS reader's hint, which echoes the exact
amount read out of the message.

## Settle Up is a money figure — treat it as one
Every expense names who it was FOR and who PAID; together they say whether one of them is
carrying the other's share. Costs for `Both` and for `AJ` split half each; a cost for one
person is owed in full by that person. Net across the period = a single figure.
`net > 0` means that person is owed. A linked reimbursement reduces the shared cost AND is
debited to whoever actually banked it — those are different people often enough to matter.
A transfer counts as a repayment only when it was recorded with the **Settlement** kind on
the Transfer form (`isSettlement`) — never inferred from the accounts, because silently
mistaking an investment move for a settlement is the worst thing this number can do. The flag
rides in an optional `is_settlement` column AND behind the description's marker
(`encodeXferDesc`), so it works with no schema change.

**Settle Up ignores the period buttons.** It is a running account from the first entry to
today and clears only when a settlement is recorded — not when the calendar turns over.

## Design Principles
- Aesthetic: premium and minimal, not flowery or decorative. Favour elegance and restraint.
- Numbers: use the clean sans-serif ('Outfit'), never a decorative serif. Apply
  `font-variant-numeric: tabular-nums` so digits stay the same width and align
  cleanly in columns/rows — figures must not visually "dance" up and down.
- The ₹ symbol and all numeric values follow the same standard sans-serif styling.
