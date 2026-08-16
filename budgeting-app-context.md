# Personal Budgeting App — Project Context & Technical Handoff

This doc is meant to be dropped into the project (e.g. repo root) and used as onboarding
context for continuing work in Claude Code, where edits happen directly against the file
rather than through chat.

## What this is

A single self-contained `index.html` file (all HTML/CSS/JS inline, no build step, no
dependencies except a Google Fonts link) implementing a personal weekly budgeting tool for
Benjamin. No frameworks, no bundler — just open the file / load the page and it runs.

## Live setup (two-repo architecture)

GitHub Pages requires a public repo, but the actual budget data needs to stay private. So
the app is split across two repos:

1. **Public repo** — contains only `index.html` (this app). GitHub Pages is enabled on it
   (Settings → Pages → Deploy from branch → `main` / root). Live at
   `https://billybobbagins.github.io/<repo-name>/`. Contains no personal data, no token —
   completely safe to be public.
2. **Private repo** (`billybobbagins/budget`) — contains only `budget-data.json`, the actual
   budget data. Read/written via the GitHub Contents API using a fine-grained PAT scoped to
   *only* this repo, with **Contents: Read and write** permission. The token is entered by
   the user into the app's Settings panel and stored only in that browser's `localStorage` —
   it is never present in the HTML file itself.

**Deployment workflow:** after editing `index.html`, it needs to be re-uploaded to the
**public** repo (overwriting the existing file) for changes to go live. GitHub Pages rebuilds
automatically, usually within about a minute. The private data repo is untouched by app-code
changes.

## Data flow / sync model

- `localStorage` (`budget_state` key) is the fast local cache — every edit saves here
  instantly, and this is what renders on load before any network call.
- The GitHub repo (`budget-data.json`) is the durable source of truth. On load (if a token is
  present), the app pulls from the repo. On every edit, it pushes (debounced ~1.1s) via PUT
  to the Contents API, using the cached `sha` for updates.
- **Merge logic**: on pull, if the local browser has never synced before on this device (no
  cached `sha` in `localStorage`), the remote is trusted unconditionally — this matters
  because a brand-new browser/origin (e.g. first time loading the Pages URL) generates a
  fresh default state stamped with the current timestamp, which would otherwise always look
  "newer" than genuinely older-but-real remote data in a naive timestamp comparison. Already
  fixed once — worth remembering if touching `pullFromGitHub()`.
- Offline: edits queue in `localStorage` regardless; an offline banner shows, and pending
  pushes retry on the `online` event.
- Manual **Export/Import JSON** buttons in Settings exist purely as an extra backup/portability
  option — not required for normal operation.

## Data model (the `state` object, what gets synced)

```js
{
  updatedAt: <ms timestamp>,
  settings: {
    startDate: '2026-08-17',   // hard-coded, Monday, Australia/Sydney — not user-editable
    initialSavings: 0,          // legacy field, no longer surfaced in UI (see below)
    weeksToShow: 10,             // how many week-cards render in the list (max 26)
    chartRange: 12               // weeks shown in the top chart/summary (independent toggle)
  },
  recurring: [
    {
      id, name, kind: 'income'|'expense',
      cadence: 'weekly'|'fortnightly'|'monthly'|'quarterly'|'yearly',
      amount, variable: bool, anchorDate: 'YYYY-MM-DD', active: bool,
      exceptions?: [{date:'YYYY-MM-DD', newDate:'YYYY-MM-DD', scope:'single'}],
      shiftFrom?: 'YYYY-MM-DD', shiftDelta?: <int days>   // "apply to all future" date edits
    }, ...
  ],
  weeks: {
    '<weekIndex>': {
      oneOffs: [{id, name, amount, kind, date: 'YYYY-MM-DD'|null}],
      overrides: { '<recurringItemId or "_rollover">': <amount> },
      skipped: [<recurringItemId>, ...]
    }, ...
  }
}
```

Week index 0 = the week starting `settings.startDate`. Weeks are computed on the fly
(`computeAll(n)` → `computeWeek(idx, priorSavings)`), not stored individually except for the
per-week overrides/one-offs/skips above — the income/expense line totals are always derived
fresh from `recurring` + that week's bucket.

## Key logic (where to look)

- **Date utils**: `fmtISO` (local calendar date, NOT `toISOString()` — that's UTC-based and
  was a real bug once, see below), `nowSydney()` (pins "today" to Australia/Sydney regardless
  of device timezone), `parseISO`, `addDays`, `addMonths`, `mondayOnOrBefore`.
- **Occurrence logic**: `rawOccurrenceDates(item, rangeStart, rangeEnd)` generates the raw
  cadence pattern; `occurrenceDatesInRange()` wraps it and applies `exceptions`/`shiftFrom`
  (date-edited occurrences); `occursInWeek()` is now just
  `occurrenceDatesInRange(...).length > 0`. All three recurring-date consumers (weekly totals,
  calendar, chart markers) go through this same path — keep it that way rather than
  reintroducing a second date-generation path.
- **Compute engine**: `computeWeek(idx, priorSavings)` → per-week income/expense lines +
  totals + savings; `computeAll(n)` chains these sequentially so rollover cascades correctly.
  Variable items use the override if set, else (for *future* weeks only) the average of past
  overrides (`avgActual()`), else the budgeted default.
- **Render**: `render()` drives both the week list (`weeksToShow`) and the chart/summary
  (`chartRange`) — these are intentionally decoupled, don't recouple them. `renderSpark()`
  draws the SVG chart with "nice" tick spacing (`niceTicks`/`niceNum`, d3-style) and markers
  for any dated items in the visible range.
- **Calendar**: `renderCalendar()` builds the month grid; `calDayClick`/`renderCalDayDetail`
  show a day's items; `calEditStart/Save/Cancel` handle in-place date editing, asking
  recurring items whether a date change is "just this occurrence" (→ `exceptions`) or "all
  future occurrences" (→ `shiftFrom`/`shiftDelta`).
- **GitHub sync**: `pullFromGitHub()`, `pushToGitHub()`, `scheduleSync()`, `manualSync()`.
  Both surface real GitHub error messages via `toast()` rather than assuming success — don't
  regress this (see bug history below).

## Constraints that are intentional, not bugs

- **Start date is hard-coded** to `2026-08-17` and has no Settings UI to change it, by
  explicit request (to avoid accidentally reshuffling the whole dataset). If this ever needs
  to change, it's a direct code edit to `defaultState()`, not a UI feature.
- **Starting savings balance** is not a Settings field — it's set by editing the "Savings"
  line directly on Week 1 (same rollover-override mechanism as any week).
- **6-month (26 week) cap** (`MAX_WEEKS` constant) applies everywhere: week list, calendar
  navigation, "Add one-off" week picker. Deliberate constraint while the feature set is still
  evolving — see "Open design questions" for what happens when this becomes limiting.
- Weeks are always Monday–Sunday, computed against Australia/Sydney time regardless of device
  timezone.

## Bug history (context so these don't get reintroduced)

1. **`fmtISO` UTC bug**: originally used `d.toISOString().slice(0,10)`, which converts to UTC
   and can shift the date backward by a day for Sydney's UTC+10/11 offset. Fixed to build the
   ISO string from local `getFullYear/getMonth/getDate()` instead.
2. **Sync icon confusion**: the "Recurring items" button originally used a circular-arrows
   icon that looked exactly like a sync/refresh icon, and there was no actual dedicated sync
   button. Fixed — sync now has its own icon+button with real toast feedback (success/specific
   error), Recurring items uses a grid icon instead.
3. **False-success sync toast**: `pullFromGitHub`'s 404-branch (repo file doesn't exist yet)
   awaited `pushToGitHub()` but showed "Created budget-data.json" unconditionally, even when
   the push had actually failed silently inside its own try/catch. Fixed by having
   `pushToGitHub()` return a boolean and surfacing the real GitHub error message.
4. **Calendar nav buttons invisible**: reused header (`.hbtn`) styling — light-colored, meant
   for the dark header — inside the light-background calendar modal, making the ‹ › buttons
   functionally invisible (not actually broken, just unreadable). Fixed with dedicated
   `.cal-nav-btn` styling.
5. **First-sync-on-new-device merge bug**: see "Data flow / sync model" above — a fresh
   browser's freshly-generated default state always out-timestamps genuinely older real data,
   so first sync on any new device must trust the remote unconditionally rather than compare
   timestamps. Fixed in `pullFromGitHub()`.

## Testing workflow used so far

Since there's no build step, "testing" so far has meant: extract the `<script>` contents and
run `node --check` on it to catch syntax errors before shipping (this environment doesn't have
a browser/DOM available to fully execute it). Logic-heavy changes (e.g. the date
exception/shift math) were additionally unit-tested by copying the relevant functions into a
standalone Node script and asserting on output. Worth continuing this pattern for anything
touching date/occurrence logic, since it's easy to get subtly wrong.

## Open design questions

- **Recurring Items header icon**: current icon (2×2 grid) isn't very indicative of
  "recurring items." Needs a better icon — revisit when a good idea comes up.
- **Recurring vs one-off visual distinction**: no current visual differentiation between
  recurring and one-off line items in a week's expense list (removed the "this week only"
  hint text per request). Not yet solved.
- **Long-term start-date drift**: because weeks never disappear and the start date is fixed,
  extended real-world use will eventually mean scrolling through a lot of past weeks to reach
  "now," and/or hitting the 6-month cap. No decision made yet on raising the cap vs. adding a
  "roll the start date forward" feature.

## Background: the original manual process (why the app works the way it does)

Benjamin previously managed budgets manually in Microsoft OneNote — copying a base week's
income/expenses and editing per-week to project forward. Paid **fortnightly, budgeted
weekly** (alternating weeks have a fortnightly income injection).

Each week conceptually has: **Balance** (recurring income + savings rollover from the
previous week), **Expenses** (itemized, each tagged with a cadence — weekly/monthly/
quarterly/one-off), and **Savings (end of week)** = Balance − Expenses, which becomes next
week's rollover. A worked 4-week example with real numbers was used to validate the app's
calculation logic during development — variable costs (e.g. groceries $150 → $200 one week)
were the specific case that motivated the "budgeted vs. actual" variable-item mechanism.

### Original core requirements (all implemented)

1. Weekly budget structure (Balance / Expenses / Savings rollover) ✅
2. Recurring fixed fees at various cadences, auto-populated ✅
3. Recurring *variable* budget categories with separate budgeted-vs-actual tracking,
   averaging into future projections ✅
4. One-off costs, any week, optionally tied to a specific date ✅
5. Automatic projection forward using fixed defaults + averaged variable costs ✅
6. Stay editable/modular — nothing locked down prematurely ✅ (ongoing philosophy)
7. Savings-account transfers treated as a plain expense line for now — deliberately still
   out of scope, no dedicated feature exists.

## Status

Core app is built and working: recurring items (fixed + variable), one-offs (week-level and
date-specific), automatic projection with averaging, editable savings rollover, calendar view
with date editing (single-occurrence vs. all-future), GitHub-backed sync across two repos
(public code / private data), 6-month cap, Sydney-timezone-correct date handling. Currently in
active user testing (Benjamin, from the live GitHub Pages deployment). Known remaining open
items are listed above under "Open design questions."
