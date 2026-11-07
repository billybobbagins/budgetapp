# Benny's Budget

Personal weekly budgeting app. **One user, one device (their iPhone, saved to home screen).** Built for
the owner only — don't design for multiple users or devices.

## Stack & layout
- Everything is in `index.html`: plain HTML + CSS + vanilla JS, no build step, no dependencies
  (Google Fonts only). Keep it that way unless asked.
- State lives in `localStorage` (`budget_state`) and syncs to `budget-data.json` in a separate
  GitHub repo (default `billybobbagins/budget`) via the Contents API, using a fine-grained PAT entered in
  Settings (stored only in localStorage).
- Device-local only, never synced: GitHub owner/repo/token/sha, PIN hash (`budget_pin_hash`),
  session unlock flag (`sessionStorage.budget_unlocked`).

## Conventions
- **Every deploy-worthy change bumps `APP_VERSION` and adds a `CHANGELOG` entry** (top of the
  `<script>`). Notes are written for the user, in plain language. Commit messages: `vX.Y.Z: summary`.
- Australian conventions: AUD (`fmtMoney`, whole dollars), `en-AU` date formatting, metric.
- "Today" is always resolved in Australia/Sydney time (`nowSydney()`), whatever timezone the device is in.
- **Date maths:** dates are local calendar dates (`parseISO`/`fmtISO`, never UTC). Get day/week
  differences only through `daysBetween()` / `weekIndexOf()`. Flooring raw millisecond differences
  breaks across daylight-saving changes (this caused the "This week" slip fixed in v2.0.0).
- Mutating a week → `weekBucket(idx)` (creates it). Reading/rendering → `readBucket(idx)` (doesn't).
- Every state change goes through `persist()` (save → mark unsynced → render → debounced push).
- Data shape changes must be handled in `normalizeState()` (runs on local load, GitHub pull and
  import) and bump `schemaVersion`.

## Data model (schemaVersion 2)
```
state = {
  schemaVersion, updatedAt,
  settings: { startDate:'2026-08-17' (fixed), initialSavings, weeksToShow (from This week), chartRange, chartEndDate },
  recurring: [{ id, name, kind:'income'|'expense', cadence:'weekly'|'fortnightly'|'monthly'|'quarterly'|'yearly',
                amount, variable (= "adjustable per week"), anchorDate, endDate|null, timing:'date'|'spread', active,
                shifts:[{from, delta}]           // "move all future occurrences", applied in order
                exceptions:[{date,newDate,scope:'single'}] }], // "move just this occurrence"
  weeks: { [idx]: { oneOffs:[{id,name,amount,kind,date|null}], overrides:{itemId|'_rollover': amt},
                    skipped:[itemId], order:[id], confirmed?:true } }
}
```
Week `idx` counts weeks from the fixed start date, Mon 17 Aug 2026 (week 0).

## Money model (`computeWeek`)
- `carriedOver` = previous week's `endBalance` (week 0: `initialSavings`). It can be overridden through
  `overrides._rollover`; week 0's override is how the real starting balance is set.
- Income / Expenses are only the week's real flows. `savings = income − expenses` (that week's net).
  `endBalance = carriedOver + savings`.
- Recurring amount for the week = override if one is set, otherwise `amount × number of occurrences that week`.
  Adjustable items always default to the standard amount. **No averaging and no avg/budget hints,
  by design.**
- Fixed (non-adjustable) recurring items are locked in week cards; they can only be skipped.
- Undated one-offs count on the **Sunday** of their week.
- `timing:'spread'` items (e.g. Spending, a weekly budget not tied to a day) are drawn on the chart as
  a straight-line daily drawdown across their week, and appear as an "Across this week" strip in the calendar.
- Status: `projected` (no edits) → `planned` (any override/one-off/skip) → `confirmed` (user tapped
  Confirm week). Editing a confirmed week keeps it confirmed. The user changes plans often, so don't nag.

## UI behaviour
- Home starts at **This week**. Past weeks sit behind a collapsed "Show N past weeks" toggle
  (newest first, still editable). Rolling planning window: `HORIZON_WEEKS` (26) from This week onward.
- The week card header shows Saved (net) and Balance (end of week). The body has a Carried over line,
  Income / Expenses boxes with their totals in the header row, and a Savings box with only a header row.
- No date tags under line items; dates are shown in the calendar.
- Tap a one-off's name to edit it (name/type/amount/date). Changing the date moves it to the right week.
- Chart: one dot per day (same-day items grouped, and the tooltip lists them all). The line is dashed
  for projected weeks.
- Drag-reorder through a single delegated `pointerdown` handler. iOS needs the non-passive
  `pointermove` with `preventDefault()` (see v1.4.2).

## Design decisions (settled — don't reopen)
- Single user, single device: **no multi-device conflict handling/merging**. GitHub sync is
  backup plus last-write-wins, and that's fine.
- The PIN is a casual deterrent (SHA-256 in localStorage) and locks once per session, not real security.
- Start date is fixed and not editable.

## Backlog / ideas
- Spending insights from past weeks (archived weeks are kept for this). The user will think about
  how to review "planned vs actual" for adjustable items. Note that overrides overwrite the plan,
  so there's no record yet of the original planned amount versus what was actually spent.
- The savings chart has been generally confusing (logged in v1.4.0). v2.0.0's money model and
  spread drawdown should help; revisit once the user has used it.
- Possibly a fuller "confirm" workflow later. It must stay flexible to on-the-fly changes.
