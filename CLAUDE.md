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

## Pickers show every option — no scrolling to find one
Long lists (category, income source, accounts) open as a tile grid inside the searchable
picker: the six you use most, learned from the log via `pickerFrequentValues()`, then the rest.
Short lists (type, frequency) are chip rows in the form itself and open nothing at all. In both
cases the `<select>` stays in the DOM as the single source of truth — the tiles and chips write
to it and dispatch `change`, so every existing handler, the SMS prefill and the edit modal keep
working untouched. A new picker follows the same rule: add the id to `PICKER_GRID_IDS`, or give
the select a `data-inline` host, rather than hand-rolling a control.

## The Log has two rows of controls, not five
Kind (All / Expense / Income / Transfer) is a segmented control on the page because it changes
every session. Everything else — person, type, category, added-by, and the bin — lives in one
bottom sheet behind a Filters button that carries a count, and whatever is on shows underneath
as a chip you can tap to drop. Account, account-type and period drill-downs from the Summary
are chips too, not banners. Sort (`logSort`, module-level so a reload starts at newest-first)
is the second button. A new filter belongs in the sheet and in `activeLogFilters()`; it does
not get a row of its own.

## Numbers are Indian — always
Grouping is Indian (`toLocaleString('en-IN')` → `12,34,567`), and compact amounts use the
Indian scale: **thousand → lakh → crore**. A figure of a lakh or more is never scaled in
thousands: `₹1.37L`, never `₹137k`. Every compact display goes through `fmtShort()`; don't
hand-roll `/1000` in a template. Full figures are whole rupees (`Math.round`) — paise are
noise at these sizes; the one exception is the SMS reader's hint, which echoes the exact
amount read out of the message.

## The SMS reader learns from corrections
Three maps in `S`, all written at save time from what the user actually chose, all beating the
built-in guesses: `smsCards` (card last-4 → account), `smsPayees` (NEFT remitter → income
source) and `smsMerchants` (merchant → {category, type}). Never ask the user to re-teach
something they have already corrected once — when a value came from one of these maps, drop
the "check this" nudge. Add new learning the same way rather than growing the hardcoded
tables.

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

**Every line of the breakdown is checkable.** The card is a balance sheet: rows are whose cost
it was, columns are who paid (`settleUp()` returns `claim`). A cell holds the other person's
share, with the spending it came out of underneath, and taps through to the Log with the filters
that produced it — kind, Paid by, For whom — so the figure can be counted against the entries.
Each column adds to its "Owed to" total, and the two totals differ by the balance. Keep it that way: a line nobody can verify does not belong on the card,
and neither does a paragraph explaining the maths in place of showing it.

## A figure that goes somewhere says so
Half the numbers on the Summary drill into the Log and half don't, and they used to look
identical. One cue, app-wide: a figure inside a row that carries a tap handler gets a hairline
underline in `--ink3` (and the row dims while pressed). Controls — buttons, chips, pills, tiles,
the segmented control — already read as controls and stay unmarked. The rule keys off
`[onclick]` in CSS, so a new drill-down inherits the cue for free; if you add one, put the
figure in the element the selector list already covers rather than inventing a marker.

## Design Principles
- Aesthetic: premium and minimal, not flowery or decorative. Favour elegance and restraint.
- **One palette, dark only.** `:root` is **Vault** — warm graphite, ivory ink, brass. There is
  no light theme and no theme toggle: don't add one back, and don't hard-code a colour in a
  rule. Every component reads from the tokens.
- **Depth, not outlines.** Cards carry a surface and a soft shadow — no 1px fences, no tinted
  boxes inside boxes. Rows inside a card are separated by `--divider` hairlines, sections by
  whitespace and a small-caps label.
- **Spend the colour once.** Brass (`--amber`) is the accent and marks selection; green and red
  mean money in and money out and nothing else. Person colours (blue / pink / teal) appear only
  as tints behind a name, never as a saturated slab.
- **Type.** 'Outfit' throughout, loaded from Google Fonts. One large light figure per card
  (weight 200–300), labels at 9.5px uppercase with 0.17–0.2em tracking, everything between
  quiet. Apply `font-variant-numeric: tabular-nums` so digits stay the same width and align
  cleanly in columns/rows — figures must not visually "dance" up and down.
- **Icons are line icons.** Inline SVG at 1.6px stroke, `currentColor`. No emoji in the UI —
  they render as another vendor's artwork and undo everything above.
- The ₹ symbol and all numeric values follow the same standard sans-serif styling.
