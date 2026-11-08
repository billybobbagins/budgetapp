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
                amount (base standard, per occurrence), amountHistory:[{week, amount}] ("from week N on" standard changes),
                anchorDate, endDate|null, timing:'date'|'spread', active,
                variable (legacy, ignored since v2.1 — every item can be changed per week),
                shifts:[{from, delta}]           // "move all future occurrences", applied in order
                exceptions:[{date,newDate,scope:'single'}] }], // "move just this occurrence"
  weeks: { [idx]: { oneOffs:[{id,name,amount,kind,date|null}], overrides:{itemId|'_rollover': amt},
                    skipped:[itemId], done:[id] (ticked paid/received), order:[id], confirmed?:true } }
}
```
Week `idx` counts weeks from the fixed start date, Mon 17 Aug 2026 (week 0).

## Money model (`computeWeek`): input → output → remainder
- **Income box = input**: `carriedOver` (previous week's remainder, or `initialSavings` for week 0,
  overridable through `overrides._rollover`) plus that week's income. `incomeTotal` includes the carried amount.
- **Expenses box = output.** **Savings box = remainder** = `incomeTotal − expenseTotal` = `endBalance`,
  which carries into the next week. Purple, always. `net` (income − expenses without the carry) is
  only used for the summary's "accrued" figure.
- The recurring amount for a week = the override if set, otherwise `standardAmount(item, idx) × occurrences that week`.
  `standardAmount` uses `amountHistory`. "This week and all future weeks" (and changing the amount
  in Recurring items) calls `setStandardFrom(item, week)` and clears that item's overrides from that week on.
  **Past weeks are never rewritten.** No averaging and no avg/budget hints, by design.
- A remainder below $0 is allowed while planning but flagged: a red "Short $X" pill, a note in the
  Savings box, and a banner on the summary pointing to the first short week (from This week on).
- Ticks (`done`) mark items paid or received. "In hand now" = carried + ticked income − ticked
  expenses. This should match the bank balance mid-week. If it doesn't, the user edits the carried-over line.
- Undated one-offs count on the **Sunday** of their week.
- `timing:'spread'` items (e.g. Spending) count 1/7 per day on the chart, and appear as an "Across this
  week" strip in the calendar.
- Status: `projected` (no edits) → `planned` (any override/one-off/skip/tick) → `confirmed` (user
  tapped Confirm week). Editing a confirmed week keeps it confirmed. The user changes plans often, so don't nag.

## UI behaviour
- Home starts at **This week**. Past weeks sit behind a collapsed "Show N past weeks" toggle
  (newest first, still editable). Rolling planning window: `HORIZON_WEEKS` (26) from This week onward.
- The week card header shows "Left" (the remainder). The body has the Income box (carried-over line
  first, then recurring items, then a "One-offs" sub-group), Expenses (recurring, then one-offs),
  a header-only purple Savings box, then a two-line add row (amount + name / type, date, Add) and Confirm week.
- Row: drag handle · tick · name + amount · ✕ (skip for recurring items, remove for one-offs).
  **Tapping a row opens the edit sheet** (`openLineEditor`). There is no inline $ editing. A recurring item
  changed for that week shows a subtle purple dot. Reset lives in the sheet, not on the row.
  Skipped items show a "Restore" link. Each sub-group is its own drag container.
- Undo: `persist()` pushes the previous snapshot onto an in-memory stack (this session only, 50 deep).
  Header button, plus an Undo link on the toast after deleting or skipping. UI-only changes use
  `persist({undo:false})`.
- No date tags under line items; dates are shown in the calendar.
- Chart (`buildTimeline` + `drawChart`, drawn in real pixel units so dots stay round): solid grey line
  through planned/confirmed weeks, dashed through projected. One dot per day: green = income,
  red = expense, split green|red = both. Dot size scales with zoom. Tapping the summary chart
  opens the interactive timeline (whole history + window, pinch/−/+ zoom down to day labels, tap a
  dot for a pop-up listing that day's items and the savings after).
- Drag-reorder through a single delegated `pointerdown` handler. iOS needs the non-passive
  `pointermove` with `preventDefault()` (see v1.4.2).

## Design decisions (settled — don't reopen)
- Single user, single device: **no multi-device conflict handling/merging**. GitHub sync is
  backup plus last-write-wins, and that's fine.
- The PIN is a casual deterrent (SHA-256 in localStorage) and locks once per session, not real security.
- Start date is fixed and not editable.

## Backlog / ideas
- **Dynamic "live week" workflow (in progress, keep brainstorming with the user).** Before v2.1 the
  user's workaround was to delete income once received, fold it into the carried savings, and delete
  expenses as they were paid, so the savings figure tracked reality. v2.1's ticks plus "In hand now"
  are the first pass at replacing that. Get feedback before going further (e.g. a reconcile-to-bank
  action, or auto-ticking dated items once their day has passed).
- Spending insights from past weeks (archived weeks are kept for this). The user will think about
  how to review "planned vs actual". Per-week overrides overwrite the plan, so there's no record yet of
  the original planned amount versus what was actually spent.
- Possibly a fuller "confirm" workflow later. It must stay flexible to on-the-fly changes.
