# Habit Tracker: Spec

## 1. Goal and constraints
- One file: `index.html` (inline `<style>` and `<script>`). Vanilla HTML/CSS/JS only.
- No libraries, no CDN, no web fonts, no build step, no network requests (no fetch), no ES modules.
- Must work by double-clicking the file (file://) in current Chrome, Edge, Firefox and Safari.
- Create no files other than `index.html` (`SPEC.md` already exists).
- Do not install packages, start servers, or run shell commands other than reading/editing files. I will open the file in the browser myself.
- Code style: `"use strict"`, small named functions, const/let, one top-level `state` variable, comments only where logic is non-obvious, no console.log.

## 2. Working rules (apply to every phase)
1. Before writing code, read this whole file, and `index.html` if it exists.
2. Do ONLY the phase I name. Never start the next phase on your own.
3. Phase 1 creates `index.html`. In later phases use targeted edits to the existing file. Never rewrite the whole file and never paste code into the chat.
4. Do not change behavior or data formats defined in this spec. Do not add anything from section 14 ("Out of scope") or any feature not listed for the current phase.
5. If something is ambiguous or two rules conflict, STOP and ask me ONE short question. Do not assume.
6. When the phase is done, reply in exactly this format, then STOP (no more tool calls, no more code):
   - What changed: max 3 lines
   - Test checklist: numbered steps I can do in the browser (max 8)
   - Final line: "Phase N complete. Reply 'confirm' to start Phase N+1, or tell me what to fix."
7. If I report a bug, fix only that bug with a minimal edit, then reply again in the rule-6 format.
8. After I reply "confirm", start the next phase from section 13.

## 3. Data model
- localStorage key `"habitTracker.v1"` holds JSON: `{ "habits": [ ... ] }`
- Each habit has:
  - `id`: string, `Date.now().toString(36) + Math.random().toString(36).slice(2, 8)`
  - `name`: string, trimmed, 1 to 40 characters
  - `activeDays`: array of unique integers 0 to 6, sorted ascending, at least one. 0 = Sunday ... 6 = Saturday (same as `Date.getDay()`)
  - `createdAt`: `"YYYY-MM-DD"` local date (today when the habit is created)
  - `completions`: array of unique `"YYYY-MM-DD"` strings, sorted ascending
- Theme (Phase 3) uses a separate key `"habitTracker.theme"` with value `"light"` or `"dark"`. Missing means follow the system.
- Load inside try/catch. If the JSON is invalid or has the wrong shape, copy the raw string to key `"habitTracker.v1.corrupt"` and start with `{ habits: [] }`.
- Save inside try/catch. If storage is unavailable, keep working in memory and show a small banner (`#storage-banner`): "Storage unavailable: changes won't be saved."

## 4. Date handling (critical)
- Always use LOCAL dates. Never use `toISOString()` or any UTC method.
- `toKey(date)`: "YYYY-MM-DD" built from `getFullYear`, `getMonth()+1`, `getDate()`, zero-padded.
- `fromKey(key)`: `new Date(y, m - 1, d, 12)` (noon avoids daylight-saving edge cases).
- `addDays(key, n)`: `fromKey(key)`, `setDate(getDate() + n)`, return `toKey(result)`.
- `weekdayOf(key)`: `fromKey(key).getDay()`.
- `todayKey()`: `toKey(new Date())`. Call it fresh every time it is needed. Never cache it at page load.
- Key strings in this format compare correctly with `<`, `>` and `===`.

## 5. Streak rules
Definitions:
- A day is "scheduled" for a habit if `weekdayOf(day)` is in `habit.activeDays`. Otherwise it is an "off-day".
- A day is "done" if its key is in `habit.completions`.
- `start` = the earlier of `habit.createdAt` and `habit.completions[0]` (just `createdAt` if there are no completions).

Pure functions (they take `today` as a parameter):
- `isScheduled(habit, key)`
- `calcCurrentStreak(habit, today)`: s = 0. For d from `today` going back one day at a time until d < start:
  - if d is done -> s++
  - else if d is scheduled AND d !== today -> stop (streak broken)
  - else -> continue without changing s (off-day not done, or today not done yet)
  - Return s.
- `calcBestStreak(habit, today)` (Phase 2): run = 0, best = 0. For d from `start` forward to `today`:
  - if d is done -> run++, best = max(best, run)
  - else if d is scheduled AND d !== today -> run = 0
  - else -> continue
  - Return best.

Consequences (do not change): not having checked in today yet never breaks a streak. Off-days are skipped. A completion on an off-day still counts +1. best >= current always. Streaks are always computed, never stored.

Worked examples for reference only. DO NOT write any test code. Today = 2026-10-29 (Thursday), createdAt = 2026-10-01, ALL = every day active, MWF = Mon/Wed/Fri:
1. ALL, done 10-27, 10-28, 10-29 -> current 3
2. ALL, done 10-27, 10-28 -> current 2 (today not done does not break)
3. ALL, done 10-26, 10-28 -> current 1 (10-27 was missed)
4. MWF, done 10-23, 10-26, 10-28 -> current 3 (today is an off-day; Tue, Sat, Sun are skipped)
5. MWF, done 10-27 (Tue, off-day), 10-28 -> current 2 (off-day completion counts; Mon 10-26 missed ends it)
6. ALL, done 10-01 to 10-05 and 10-07 to 10-09 -> best 5, current 0
7. ALL, no completions -> current 0, best 0

## 6. Completion rules
`toggleCompletion(habitId, key)` does something only if `key <= todayKey()` AND (the day is scheduled for that habit OR `key` is already in `completions`, so stray off-day completions can be removed). Otherwise it does nothing.
It adds the key if missing or removes it if present, keeps the array sorted and unique, saves, and re-renders.
Future days are never togglable. Any past day, including days before `createdAt`, is togglable if scheduled (full backfill). Backfilling before `createdAt` needs no special handling: `start` in section 5 already covers it.

## 7. Code layout and element IDs
Script order (top to bottom): constants -> date helpers -> storage (load/save) -> streak functions -> actions (toggle, add, edit, delete) -> render functions -> event listeners -> init.
Init order: (Phase 3: theme is already applied by the head script) -> load state -> (Phase 3: set `lastToday`) -> attach listeners -> `render()`.

Use exactly these IDs:
- `#storage-banner`, `#theme-btn` (Phase 3), `#add-btn`
- `#stats`, `#progress` (outer track) with `#progress-fill`
- `#history-bar` with `#history-prev`, `#history-range`, `#history-next` (Phase 2)
- `#habit-grid`
- `#habit-dialog` (dialog), `#habit-form`, `#dialog-title`, `#habit-name`, `#day-chips`, `#form-error`, `#cancel-btn`, `#submit-btn`

Variable `editingId` is `null` in add mode and the habit id in edit mode.

## 8. UI structure (dashboard style)
Global: `<html lang="en">`, `<meta name="viewport" content="width=device-width, initial-scale=1">`, `<link rel="icon" href="data:,">`, `<title>Habit Tracker</title>`. Max-width 1000px centered, 16px page padding, font stack `system-ui, -apple-system, "Segoe UI", Roboto, sans-serif`, 16px base size, 12px border radius, spacing in multiples of 4px, transitions at most `.15s`. EVERY color must come from the CSS variables in section 11.

Header: title "Habit Tracker" (h1) on the left. On the right: the theme toggle (Phase 3 only) and a primary button "+ New habit" (`#add-btn`).

Stats row (`#stats`): CSS grid `repeat(auto-fit, minmax(150px, 1fr))`, gap 12px. Each stat card: small muted label above a 28px bold value:
- "Today": done/total of habits scheduled today, e.g. "2/3". Only habits scheduled today count. Show "-" if none are scheduled today.
- "Current streak": the highest current streak among all habits (0 if no habits), with the small muted sub-label "days".
- "Best streak" (Phase 2): the highest best streak among all habits (0 if none).
- "Completions" (Phase 2): total completions across all habits.

Progress bar: below the stats row, 8px high, track uses `--surface-2`, fill uses `--accent`, fill width = done/total scheduled today as a percent (0 if none scheduled). Hidden when there are no habits.

Habit grid (`#habit-grid`): CSS grid `repeat(auto-fill, minmax(280px, 1fr))`, gap 12px. Habits appear in creation order. No drag and drop.

Habit card: `--surface` background, 1px `--border`, 16px padding, `--shadow`. Rows top to bottom:
1. Name (bold, single line with ellipsis) on the left, streak badge on the right with text "N day streak" (N = current streak).
2. Muted text: "Every day" if all 7 days are active, otherwise short names in Mon to Sun order joined by spaces, e.g. "Mon Wed Fri" (short names: Mon Tue Wed Thu Fri Sat Sun).
3. (Phase 2) Three mini stats: Current / Best / Total (Total = `completions.length`).
4. A full-width "today" button (`data-action="toggle-today"`):
   - today scheduled and not done: label "Mark done"
   - today done (scheduled or not): label "Done ✓", `--accent` background with `--accent-contrast` text, `aria-pressed="true"`, clicking undoes it
   - today is an off-day and not done: disabled button labeled "Rest day"
5. (Phase 2) History grid (section 9).
6. Footer: small text buttons "Edit" (Phase 2) and "Delete" (text color `--danger`).

Add/Edit dialog: one native `<dialog id="habit-dialog">` containing a `<form id="habit-form">`. Handle submit with `preventDefault`, validate, then call `dialog.close()` manually. Contents:
- Title (`#dialog-title`): "New habit" in add mode, "Edit habit" in edit mode.
- Text input "Habit name" (`#habit-name`, maxlength 40, autofocus).
- Seven day chips in `#day-chips`, displayed in order Mon Tue Wed Thu Fri Sat Sun. Each is a real checkbox inside a `<label>` styled as a chip. Values: Mon=1, Tue=2, Wed=3, Thu=4, Fri=5, Sat=6, Sun=0. Add mode: all 7 checked by default. Checked chips use `--accent` background.
- An inline error paragraph (`#form-error`, `role="alert"`). Never use `alert()`.
- Buttons: "Cancel" (`type="button"`, closes the dialog) and the submit button labeled "Add habit" (add mode) or "Save" (edit mode).
- Validation: trimmed name not empty and at most 40 characters; no other habit with the same name ignoring case (ignore the habit being edited); at least one day checked. On failure show the message in the error paragraph and keep the dialog open. Messages: "Enter a habit name.", "That name already exists.", "Pick at least one day."
- Opening the dialog clears the previous error. Add mode creates `{ id, name, activeDays (sorted ascending numerically), createdAt: todayKey(), completions: [] }`.

Empty state: when there are no habits, show centered muted text in place of the grid: "No habits yet. Click '+ New habit' to start."

Delete: `confirm(`Delete "${habit.name}"? This removes all its history.`)`. If confirmed, remove the habit, save, render.

Rendering: one `render()` function rebuilds the stats row, progress bar and habit grid from `state` by assigning template-string HTML. ALL user text must pass through `escapeHtml()` (escape & < > " '). Use event delegation: one click listener on `#habit-grid` reads `data-action` (`toggle-today`, `toggle-day`, `edit`, `delete`) plus `data-id` and `data-date` from the closest button.

Accessibility: real `<button>` and `<input>` elements only. Toggle buttons use `aria-pressed`. Icon-only buttons have `aria-label`. Visible `:focus-visible` outline of 2px solid `var(--accent)`. Buttons are at least 40px high (history cells excepted).

## 9. History grid (Phase 2)
Each card shows a row of 14 days. A global variable `historyPage` (integer, starts at 0, not saved) selects which 14 days are shown, so any past day can be reached.
- Window: `windowEnd = addDays(todayKey(), -14 * historyPage)`. Days shown: the 14 days ending at `windowEnd`, oldest on the left. At `historyPage` 0 the right-most cell is today.
- Navigation bar `#history-bar` sits between the progress bar and the habit grid (hidden when there are no habits): button `#history-prev` "‹ Earlier", label `#history-range` showing the first and last visible dates formatted `toLocaleDateString('en-GB', { day: 'numeric', month: 'short' })` joined by " – ", and button `#history-next` "Later ›". "Earlier" increments `historyPage` (max 26, then disabled). "Later" decrements it (disabled at 0). Both re-render.
- One `<button>` per day: 24px by 24px, 6px radius, 4px gap, with the weekday initial (S M T W T F S from `weekdayOf`) in 10px muted text below the cell. On narrow screens cells may shrink (`flex: 1; max-width: 24px`).
- States:
  - done: filled with `--accent`
  - missed (scheduled, in the past, not done): empty with 1px `--border`
  - today, not done: empty with 2px `--accent` border
  - before the habit's `createdAt`, scheduled, not done: like missed but opacity .5 (still clickable)
  - off-day, not done: opacity .4, dashed border, disabled
  - off-day, done: filled like done and clickable (so it can be removed)
- Clickable (`data-action="toggle-day"` with `data-id` and `data-date`) only if scheduled or already done.
- Tooltip (`title`): `fromKey(key).toLocaleDateString('en-GB', { weekday: 'short', day: 'numeric', month: 'short' })`.

## 10. Edit habit (Phase 2)
"Edit" opens the same dialog in edit mode (`editingId` set), prefilled with the name and active days. Save updates ONLY `name` and `activeDays`. `completions` and `createdAt` are untouched. Streaks update automatically because they are computed.

## 11. Theme variables (teal accent)
Define on `:root` from Phase 1 (light):
```
--bg:#f4f7f8; --surface:#ffffff; --surface-2:#e8f0f1; --text:#16222a; --muted:#566872;
--border:#d6e0e3; --accent:#0f766e; --accent-contrast:#ffffff; --danger:#d93f3f;
--shadow:0 1px 3px rgba(16,24,40,.08);
```
Dark values (added in Phase 3):
```
--bg:#0d1517; --surface:#152023; --surface-2:#1e2d31; --text:#e6f0f1; --muted:#93a9ae;
--border:#2a3c41; --accent:#2dd4bf; --accent-contrast:#08201d; --danger:#f87171;
--shadow:0 1px 3px rgba(0,0,0,.5);
```
Phase 3 mechanics:
- Add `<meta name="color-scheme" content="light dark">`.
- Put the dark values in two places: under `:root[data-theme="dark"]`, and under `@media (prefers-color-scheme: dark) { :root:not([data-theme="light"]) { ... } }`.
- A tiny inline script in `<head>` (try/catch) reads `"habitTracker.theme"` and sets `document.documentElement.dataset.theme` before first paint.
- Header toggle button `#theme-btn`: shows 🌙 in light mode and ☀️ in dark mode, `aria-label="Toggle dark mode"`. On click: work out the current effective theme (`dataset.theme`, else `matchMedia('(prefers-color-scheme: dark)').matches`), switch to the opposite, set `dataset.theme` and save it to `"habitTracker.theme"` (in try/catch). Update the icon.
- No hard-coded colors anywhere in the CSS outside the variable definitions.

## 12. Day rollover (Phase 3)
Keep a variable `lastToday`. Every 30 seconds (`setInterval`) and on `document` "visibilitychange" when visible: if `todayKey() !== lastToday`, update `lastToday` and call `render()`.

## 13. Phases

### Phase 1: core app (create index.html)
Build: sections 3, 4, 6, 7; `isScheduled` and `calcCurrentStreak` from section 5; section 8 limited to: Header (no theme button), Stats row with only "Today" and "Current streak", Progress bar, Habit grid, Habit card rows 1, 2, 4 and 6 (Delete only), Add dialog (add mode only), Empty state, Delete, Rendering, Accessibility. Light theme variables only (section 11, light block).
Do NOT build: Edit, best streak, totals, history grid and its navigation, dark mode, theme toggle, 🔥, day rollover.
Test checklist must cover: add a Mon/Wed/Fri habit; duplicate name shows an error; empty name shows an error; no day selected shows an error; mark today done and see the streak and stats update; the "Rest day" button on an off-day; reload keeps data; delete with confirm; the empty state.

### Phase 2: depth (targeted edits only)
Build: `calcBestStreak`; stats cards "Best streak" and "Completions"; card row 3 (Current / Best / Total); card row 5 history grid with `#history-bar` paging and click-to-toggle (section 9); "Edit" button and edit mode of the dialog (section 10).
Do NOT build: dark mode, theme toggle, 🔥, day rollover, any test code.
Test checklist must cover: toggle past days in the grid and watch the streaks change; use Earlier/Later to reach older days and backfill one; backfill a missed day to repair a streak; edit a name; edit active days and see streaks recompute; reload keeps everything.

### Phase 3: polish (targeted edits only)
Build: dark mode and theme toggle (section 11); day rollover (section 12); "🔥 " before the streak badge text when current streak >= 3; Enter in the name field submits the dialog.
Then do a final review against this spec and list any deviations in max 5 lines (or say "no deviations").
Test checklist must cover: toggle persists after reload; first load follows the OS theme; no hard-coded colors; layout at about 375px wide; no console errors.

## 14. Out of scope (never add)
Export/import, emoji or color per habit, reminders or notifications, charts, drag and drop, sounds, extra animations, undo for delete, habit archiving, multiple users, accounts, any test code or `#test` hook, any additional pages or files.
