# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Stack

Pure HTML / CSS / JavaScript — no external frameworks, libraries, or build tools.
Everything lives in a single file.

## Files

| File | Description |
|---|---|
| `todo_list_pdf.html` | **Production app** — full planner with deadline, dashboard, calendar view |
| `index.html` | Original MVP — simple todo list (reference only) |
| `planner_prd.md` | Product Requirements Document |
| `planner_v1.html` | Intermediate build artifact (backend logic draft) |
| `planner_v2.html` | Intermediate build artifact (frontend UI draft) |

## Running

Open `todo_list_pdf.html` directly in a browser. No build step required.
Desktop-only (no mobile layout). Viewport fixed at `width=1024`.

## Architecture

Single-file app (`todo_list_pdf.html`). All HTML, CSS, and JS are inline.

### Todo data model
```
{
  id        : string        // crypto.randomUUID() or Date.now()+Math.random() fallback
  text      : string        // max 200 chars
  category  : '업무' | '개인' | '공부'
  completed : boolean
  createdAt : string        // ISO 8601
  deadline  : string | null // 'YYYY-MM-DD' date string, or null if not set
}
```

### Constants
- `STORAGE_KEY = 'planner-todos'` — localStorage key for todos
- `STORAGE_SORT_KEY = 'planner-todos-sort'` — localStorage key for active sort
- `VALID_CATS = ['업무', '개인', '공부']` — allowed category values
- `KEYWORD_MAP` — category → string[] keyword table used for auto-classification

### App state (module-level vars)
| Variable | Type | Purpose |
|---|---|---|
| `todos` | `Array` | Single source of truth; synced to localStorage on every mutation |
| `editingId` | `string\|null` | ID of the item currently in inline-edit mode |
| `activeFilter` | `string` | `'all'` or a category name; controls which items are rendered |
| `activeSort` | `string` | `'created'` \| `'deadline'` \| `'category'`; persisted to localStorage |
| `activeView` | `string` | `'list'` \| `'calendar'`; toggles between list and calendar view |
| `calendarYear` | `number` | Currently displayed calendar year |
| `calendarMonth` | `number` | Currently displayed calendar month (0-indexed) |
| `selectedCalDate` | `string\|null` | `'YYYY-MM-DD'` of clicked calendar day; filters list to that date |
| `selectionMode` | `boolean` | Whether bulk-select UI is active |
| `selectedIds` | `Set<string>` | IDs of currently selected items |
| `userOverrideCat` | `boolean` | `true` after user manually changes the category select; suppresses auto-classification for that input session |

### localStorage layer
- `loadTodos()` — reads from localStorage, runs `sanitize()`, returns `[]` on any failure
- `sanitize(data)` — validates shape; repairs or drops invalid items; preserves `deadline` if valid `YYYY-MM-DD`, otherwise sets to `null`
- `saveTodos(list)` — JSON-serialises to localStorage; silently swallows quota errors
- `loadSort()` / `saveSort(v)` — persists `activeSort` across sessions

### Date helpers
- `getTodayStr()` — returns today as `'YYYY-MM-DD'` in local timezone
- `daysUntil(dateStr)` — returns signed integer days from today to deadline (negative = overdue)
- `isValidDeadline(v)` — returns true if v matches `/^\d{4}-\d{2}-\d{2}$/`

### Rendering pipeline
Every state mutation ends with `saveTodos(todos)` → `renderTodos()`.

| Function | Role |
|---|---|
| `renderTodos()` | Clears `#todo-list`, applies filter + sort, builds rows, syncs filter/sort/view UI, calls `renderSelToolbar()`, `updateProgress()`, `updateDashboard()`, `renderCalendar()` |
| `buildViewRow(li, todo)` | Normal read mode: checkbox, text, deadline badge, category badge, edit/delete buttons |
| `buildSelectRow(li, todo, isSelected)` | Bulk-select mode: round checkbox, text, badges; row click toggles selection |
| `buildEditRow(li, todo)` | Inline edit mode: text input + date input + category select + save/cancel |
| `buildDeadlineBadge(deadline, completed)` | Returns a `<span>` deadline badge: `D+N 지남` (red) / `오늘 마감` (amber) / `D-N` (amber ≤3, blue >3) / neutral date if completed |
| `renderSelToolbar(visible)` | Redraws `#sel-toolbar` — "선택" button (normal) or full bulk-action bar with tristate checkbox |
| `updateProgress()` | Updates `#progress-label` and `#progress-fill`; computed from full `todos` array |
| `updateDashboard()` | Updates 4 stat cards + category progress bars in one O(n) pass over `todos` |
| `renderCalendar()` | Clears `#calendar-container`, builds nav + month summary + 7-col grid |
| `buildCalendarNav()` | Returns nav bar with ‹ prev / 오늘 / next › buttons and `#cal-title` |
| `buildCalendarGrid()` | Returns 7-col grid: header row (일~토) + day cells with task chips |
| `getCalendarDays(year, month)` | Returns 35 or 42 day objects `{date, day, inMonth, isToday, dow, todos}` starting from the Sunday of the week containing the 1st |
| `setView(v)` | Sets `activeView`, clears `selectedCalDate`, calls `renderTodos()` |
| `sortTodos(list)` | Returns sorted copy of list by `activeSort` (deadline: soonest first, nulls last; created: newest first; category: alphabetical) |

### Features

**CRUD**
- `addTodo()` — validates non-empty text, reads deadline input, pushes new item, clears inputs
- `toggleTodo(id)` — flips `completed`
- `startEdit(id)` / `saveEdit(id, text, cat, deadline)` / `cancelEdit()` — inline edit lifecycle; entering edit mode clears selection mode
- `deleteTodo(id, text)` — single-item delete with `confirm()`

**Deadline management**
- Optional `deadline` field (`YYYY-MM-DD`) set via `<input type="date" id="deadline-input">`
- `buildDeadlineBadge()` computes days remaining and returns colour-coded badge
- Overdue items get class `overdue` (red left border + light red background)
- Due-today items get class `due-today` (amber left border + light amber background)

**Dashboard**
- Always visible above the input area
- 4 stat cards: 전체 할 일 / 오늘 완료 / 마감 임박 (≤3 days) / 지연 중
- Category progress rows: 업무 / 개인 / 공부 with mini progress bars
- Updated on every `renderTodos()` call via `updateDashboard()`

**Calendar view**
- Toggle between "목록" and "달력" via `#view-toggle` pills
- Monthly grid (일~토), navigated with ‹ / 오늘 / › buttons
- Task chips on each deadline date (max 3 + "+N more")
- Today cell highlighted; overdue days have red border
- Clicking a day sets `selectedCalDate` → list below filters to that day
- Clicking same day again deselects (shows all)
- Month summary: "N개 일정" count below nav

**Sort**
- Three sort pills: 생성순 (newest first) / 마감순 (soonest deadline first, nulls last) / 카테고리순
- Sort choice persisted to localStorage via `STORAGE_SORT_KEY`

**Bulk select / delete**
- "선택" button enters `selectionMode`; items switch to `buildSelectRow`
- `deleteSelected()` — single `confirm()` for N items; auto-exits selection mode when 0 items remain
- Filter changes preserve `selectionMode` but clear `selectedIds`

**Auto category classification**
- `detectCategory(text)` — scores each category by counting `KEYWORD_MAP` hits; returns highest-scoring category or `null`
- Fires on every `input` event (120ms debounce) while `userOverrideCat === false`
- Manual `change` on `#category-select` sets `userOverrideCat = true`
- `addTodo()` and clearing the input both reset `userOverrideCat` to `false`
- `showBadge()` / `hideBadge()` toggle the `#auto-cat-badge` pill

**Filtering**
- Four filter buttons (`전체 / 업무 / 개인 / 공부`) stored in `activeFilter`; `getVisible()` applies the filter
- When `selectedCalDate` is set in calendar view, `getVisible()` additionally filters by that date

### CSS design tokens (`:root`)
`--primary`, `--primary-dk`, `--primary-pale`, `--danger`, `--danger-bg`, `--success`, `--success-bg`, `--warn`, `--warn-bg`, `--sel-color`, `--bg`, `--surface`, `--border`, `--border-subtle`, `--text`, `--text-sub`, `--text-muted`, `--r`, `--r-sm`, `--r-pill`, `--shadow`, `--shadow-up`, `--shadow-card`, `--tr`, `--deadline-soon`, `--deadline-overdue`, `--deadline-ok`, `--card-bg`, `--card-shadow`, `--dash-border`, `--cal-cell-bg`, `--cal-cell-hover`, `--cal-cell-radius`

No responsive/mobile breakpoint. Desktop-only layout.
