# Design Document: Todo-Life Dashboard

## Overview

The Todo-Life Dashboard is a self-contained, client-side productivity application delivered as three files: `index.html`, `css/style.css`, and `js/app.js`. There is no build step, no server, and no external dependencies. All state is held in memory during a session and persisted to `localStorage` between sessions.

The application is composed of four independent UI widgets rendered inside a single page:

| Widget | Purpose |
|---|---|
| Greeting Widget | Live clock, date, and time-of-day greeting |
| Focus Timer | 25-minute Pomodoro countdown |
| To-Do List | Task CRUD with localStorage persistence |
| Quick Links | Labelled URL shortcuts with localStorage persistence |

Each widget is self-contained: it owns its DOM subtree, its state, and its localStorage key (where applicable). Widgets communicate only through shared utility functions (storage helpers, DOM helpers) — never directly with each other.

---

## Architecture

### High-Level Structure

```
index.html          ← single HTML shell; loads css/style.css and js/app.js
css/
  style.css         ← all styles; single file, no preprocessor
js/
  app.js            ← all JavaScript; single file, no frameworks
```

### JavaScript Architecture

`app.js` is organised as a collection of immediately-invoked module-like IIFE blocks and plain functions, grouped by concern. There is no module bundler; the file is loaded with a `<script defer src="js/app.js">` tag.

```
app.js
├── CONFIG            — constants (timer duration, storage keys, greeting thresholds)
├── StorageService    — read/write/error-handling wrapper around localStorage
├── GreetingWidget    — clock/date/greeting logic
├── FocusTimer        — countdown state machine
├── TodoList          — task CRUD + render
├── QuickLinks        — link CRUD + render
└── init()            — bootstraps all widgets on DOMContentLoaded
```

Each widget section follows the same internal pattern:

1. **State object** — plain JS object holding the widget's current data
2. **Render function** — reads state, rebuilds the relevant DOM subtree
3. **Event handlers** — mutate state, call render, call StorageService
4. **init function** — loads persisted data, attaches event listeners, calls render

### Execution Flow

```
DOMContentLoaded
  └── init()
        ├── GreetingWidget.init()   → start 1-min interval
        ├── FocusTimer.init()       → attach button listeners
        ├── TodoList.init()         → load from localStorage, render
        └── QuickLinks.init()       → load from localStorage, render
```

---

## Components and Interfaces

### 1. CONFIG

```js
const CONFIG = {
  TIMER_DURATION_SECONDS: 25 * 60,   // 1500
  STORAGE_KEY_TASKS: 'todo_life_tasks',
  STORAGE_KEY_LINKS: 'todo_life_links',
  GREETING_THRESHOLDS: {
    MORNING:   { start: 5,  end: 12 },  // 05:00–11:59
    AFTERNOON: { start: 12, end: 18 },  // 12:00–17:59
    EVENING:   { start: 18, end: 22 },  // 18:00–21:59
    // NIGHT: everything else (22:00–04:59)
  },
};
```

### 2. StorageService

Wraps `localStorage` with try/catch so widget code never needs to handle storage errors directly.

```js
const StorageService = {
  /**
   * Read and JSON-parse a value. Returns null if key absent or parse fails.
   * @param {string} key
   * @returns {any|null}
   */
  load(key) { ... },

  /**
   * JSON-stringify and write a value.
   * On error, fires a non-blocking warning (console.warn + optional UI toast).
   * @param {string} key
   * @param {any} value
   * @returns {boolean} true on success, false on failure
   */
  save(key, value) { ... },

  /**
   * Returns true if localStorage is available and writable.
   * @returns {boolean}
   */
  isAvailable() { ... },
};
```

### 3. GreetingWidget

**DOM target:** `#greeting-widget`

**State:** No persistent state. Reads `new Date()` on each tick.

**Public interface:**
```js
const GreetingWidget = {
  init()  { ... },   // starts setInterval(tick, 60_000), calls tick() immediately
  tick()  { ... },   // reads clock, updates #greeting-text, #time-display, #date-display
};
```

**Greeting logic:**
```js
function getGreeting(hour) {
  if (hour >= 5  && hour < 12) return 'Good Morning';
  if (hour >= 12 && hour < 18) return 'Good Afternoon';
  if (hour >= 18 && hour < 22) return 'Good Evening';
  return 'Good Night';
}
```

**Time formatting:**
- Time: `HH:MM` using `Date#toLocaleTimeString` with `{ hour: '2-digit', minute: '2-digit', hour12: false }`
- Date: `Date#toLocaleDateString` with `{ weekday: 'long', day: 'numeric', month: 'long', year: 'numeric' }`

### 4. FocusTimer

**DOM target:** `#focus-timer`

**State:**
```js
let timerState = {
  remaining: CONFIG.TIMER_DURATION_SECONDS,  // seconds left
  intervalId: null,                           // null when stopped
  isRunning: false,
};
```

**Public interface:**
```js
const FocusTimer = {
  init()   { ... },  // attach click listeners to #btn-start, #btn-stop, #btn-reset
  start()  { ... },  // sets interval, disables start button
  stop()   { ... },  // clears interval, enables start button
  reset()  { ... },  // stop + restore remaining to 1500
  tick()   { ... },  // decrement remaining, update display, check for 00:00
  render() { ... },  // format remaining as MM:SS, write to #timer-display
};
```

**State machine:**

```
IDLE ──[start]──► RUNNING ──[stop]──► PAUSED ──[start]──► RUNNING
  ▲                  │                   │
  └──────[reset]─────┘                   │
  └──────────────────[reset]─────────────┘
  ◄──────────────[reaches 00:00]──────────
```

When the timer reaches 00:00, it transitions to a FINISHED state: the display shows "00:00", a `.timer-finished` CSS class is applied to `#focus-timer`, and the start button remains disabled until reset.

### 5. TodoList

**DOM target:** `#todo-list`

**State:**
```js
let tasks = [];  // Task[]
```

**Task schema** (see Data Models section).

**Public interface:**
```js
const TodoList = {
  init()              { ... },  // load from storage, render, attach form listener
  addTask(text)       { ... },  // create Task, push to tasks[], save, render
  editTask(id, text)  { ... },  // find by id, update text, save, render
  toggleTask(id)      { ... },  // flip completed, save, render
  deleteTask(id)      { ... },  // filter out by id, save, render
  render()            { ... },  // rebuild #task-list UL from tasks[]
  save()              { ... },  // StorageService.save(STORAGE_KEY_TASKS, tasks)
};
```

**Render strategy:** Full re-render on every mutation. The `render()` function clears `#task-list` and rebuilds it from the `tasks` array. This is acceptable for the expected data size (tens to low hundreds of tasks).

**Edit mode:** When a task enters edit mode, its `<li>` element is replaced with an inline `<input>` + confirm/cancel buttons. Only one task can be in edit mode at a time (activating edit on a second task auto-cancels the first).

### 6. QuickLinks

**DOM target:** `#quick-links`

**State:**
```js
let links = [];  // Link[]
```

**Link schema** (see Data Models section).

**Public interface:**
```js
const QuickLinks = {
  init()          { ... },  // load from storage, render, attach form listener
  addLink(label, url) { ... },  // create Link, push, save, render
  deleteLink(id)  { ... },  // filter out by id, save, render
  render()        { ... },  // rebuild #links-grid from links[]
  save()          { ... },  // StorageService.save(STORAGE_KEY_LINKS, links)
};
```

---

## Data Models

### Task

```js
/**
 * @typedef {Object} Task
 * @property {string}  id        - Unique identifier (crypto.randomUUID() or Date.now().toString())
 * @property {string}  text      - Task description (non-empty, trimmed)
 * @property {boolean} completed - Completion status; false by default
 * @property {number}  createdAt - Unix timestamp (ms) of creation
 */
```

**localStorage representation** (`todo_life_tasks`):
```json
[
  { "id": "1716739200000", "text": "Write design doc", "completed": false, "createdAt": 1716739200000 },
  { "id": "1716739260000", "text": "Review PR",        "completed": true,  "createdAt": 1716739260000 }
]
```

### Link

```js
/**
 * @typedef {Object} Link
 * @property {string} id    - Unique identifier
 * @property {string} label - Display label (non-empty, trimmed)
 * @property {string} url   - Target URL (non-empty; opened with window.open)
 */
```

**localStorage representation** (`todo_life_links`):
```json
[
  { "id": "1716739300000", "label": "GitHub",  "url": "https://github.com" },
  { "id": "1716739360000", "label": "MDN Docs", "url": "https://developer.mozilla.org" }
]
```

### Validation Rules

| Field | Rule |
|---|---|
| `Task.text` | Non-empty after `String.prototype.trim()` |
| `Task.completed` | Boolean; defaults to `false` |
| `Link.label` | Non-empty after trim |
| `Link.url` | Non-empty after trim; no further URL validation required |

---

## HTML Layout

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Todo-Life Dashboard</title>
  <link rel="stylesheet" href="css/style.css">
</head>
<body>
  <div id="storage-warning" class="storage-warning hidden" role="alert" aria-live="polite"></div>

  <main class="dashboard">

    <!-- Widget 1: Greeting -->
    <section id="greeting-widget" class="widget widget--greeting" aria-label="Greeting and time">
      <p id="greeting-text" class="greeting__text"></p>
      <p id="time-display"  class="greeting__time"></p>
      <p id="date-display"  class="greeting__date"></p>
    </section>

    <!-- Widget 2: Focus Timer -->
    <section id="focus-timer" class="widget widget--timer" aria-label="Focus timer">
      <h2 class="widget__title">Focus Timer</h2>
      <p id="timer-display" class="timer__display" aria-live="polite" aria-atomic="true">25:00</p>
      <div class="timer__controls">
        <button id="btn-start" class="btn btn--primary">Start</button>
        <button id="btn-stop"  class="btn btn--secondary">Stop</button>
        <button id="btn-reset" class="btn btn--ghost">Reset</button>
      </div>
    </section>

    <!-- Widget 3: To-Do List -->
    <section id="todo-list" class="widget widget--todo" aria-label="To-do list">
      <h2 class="widget__title">To-Do</h2>
      <form id="todo-form" class="todo__form" novalidate>
        <input id="todo-input" type="text" class="input" placeholder="Add a task…" aria-label="New task text" autocomplete="off">
        <button type="submit" class="btn btn--primary">Add</button>
      </form>
      <ul id="task-list" class="task-list" aria-label="Task list"></ul>
    </section>

    <!-- Widget 4: Quick Links -->
    <section id="quick-links" class="widget widget--links" aria-label="Quick links">
      <h2 class="widget__title">Quick Links</h2>
      <form id="links-form" class="links__form" novalidate>
        <input id="link-label-input" type="text"  class="input" placeholder="Label"  aria-label="Link label">
        <input id="link-url-input"   type="url"   class="input" placeholder="URL"    aria-label="Link URL">
        <button type="submit" class="btn btn--primary">Add</button>
      </form>
      <div id="links-grid" class="links__grid" aria-label="Saved links"></div>
    </section>

  </main>

  <script defer src="js/app.js"></script>
</body>
</html>
```

---

## CSS Architecture

### File: `css/style.css`

The stylesheet is organised into clearly commented sections:

```
1. CSS Custom Properties (design tokens)
2. Reset / Base
3. Layout — .dashboard grid
4. Widget — shared .widget card styles
5. Greeting Widget
6. Focus Timer
7. To-Do List
8. Quick Links
9. Buttons
10. Form inputs
11. Utility classes (.hidden, .sr-only, etc.)
12. Responsive breakpoints
```

### Design Tokens (CSS Custom Properties)

```css
:root {
  /* Colour palette */
  --color-bg:          #f5f5f0;
  --color-surface:     #ffffff;
  --color-border:      #e0e0d8;
  --color-text-primary:   #1a1a1a;
  --color-text-secondary: #6b6b6b;
  --color-accent:      #4a7c59;   /* green — primary actions */
  --color-accent-hover:#3a6347;
  --color-danger:      #c0392b;
  --color-danger-hover:#a93226;
  --color-finished:    #e8f5e9;   /* timer finished background */

  /* Typography */
  --font-family:  system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif;
  --font-size-xs: 0.75rem;   /* 12px */
  --font-size-sm: 0.875rem;  /* 14px */
  --font-size-md: 1rem;      /* 16px */
  --font-size-lg: 1.25rem;   /* 20px */
  --font-size-xl: 2rem;      /* 32px */
  --font-size-2xl: 3rem;     /* 48px — timer display */

  /* Spacing scale */
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 0.75rem;
  --space-4: 1rem;
  --space-6: 1.5rem;
  --space-8: 2rem;

  /* Borders */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;

  /* Shadows */
  --shadow-card: 0 1px 4px rgba(0,0,0,0.08);
}
```

### Layout

The dashboard uses CSS Grid with a responsive column strategy:

```css
.dashboard {
  display: grid;
  gap: var(--space-6);
  padding: var(--space-6);
  max-width: 1200px;
  margin: 0 auto;

  /* Mobile-first: single column */
  grid-template-columns: 1fr;
}

/* ≥ 640px: two columns */
@media (min-width: 640px) {
  .dashboard {
    grid-template-columns: repeat(2, 1fr);
  }
}

/* ≥ 1024px: four columns */
@media (min-width: 1024px) {
  .dashboard {
    grid-template-columns: repeat(4, 1fr);
  }
  /* Greeting spans 2 cols, Todo spans 2 cols */
  .widget--greeting { grid-column: span 2; }
  .widget--todo     { grid-column: span 2; }
}
```

### Responsive Behaviour Summary

| Viewport | Layout |
|---|---|
| 320px – 639px | 1 column, widgets stacked vertically |
| 640px – 1023px | 2 columns |
| 1024px – 1920px | 4 columns; Greeting and Todo span 2 each |

### Typography

- Base font size: `16px` on `<html>` (browser default)
- Body text / task labels: minimum `var(--font-size-sm)` = 14px
- Timer display: `var(--font-size-2xl)` = 48px, `font-weight: 700`, `font-variant-numeric: tabular-nums`
- Greeting time: `var(--font-size-xl)` = 32px
- Widget titles: `var(--font-size-lg)` = 20px, `font-weight: 600`

### Colour Contrast

All text/background pairings are chosen to meet WCAG 2.1 AA (≥ 4.5:1 for normal text, ≥ 3:1 for large text):

| Element | Text | Background | Approx. ratio |
|---|---|---|---|
| Body text | `#1a1a1a` | `#ffffff` | 18.1:1 ✓ |
| Secondary text | `#6b6b6b` | `#ffffff` | 5.7:1 ✓ |
| Accent button text | `#ffffff` | `#4a7c59` | 4.6:1 ✓ |
| Completed task text | `#6b6b6b` | `#ffffff` | 5.7:1 ✓ |

### Task Visual States

```css
.task-item--completed .task-item__text {
  text-decoration: line-through;
  color: var(--color-text-secondary);
}

.widget--timer.timer-finished {
  background-color: var(--color-finished);
}
```

---

## Event Handling Patterns

### Pattern 1: Form Submit (Add Task / Add Link)

```js
document.getElementById('todo-form').addEventListener('submit', (e) => {
  e.preventDefault();
  const text = document.getElementById('todo-input').value.trim();
  if (!text) {
    document.getElementById('todo-input').focus();
    return;
  }
  TodoList.addTask(text);
  document.getElementById('todo-input').value = '';
  document.getElementById('todo-input').focus();
});
```

### Pattern 2: Event Delegation (Task Actions)

Rather than attaching listeners to each task button, a single listener on `#task-list` handles all task interactions via `data-*` attributes:

```js
document.getElementById('task-list').addEventListener('click', (e) => {
  const btn = e.target.closest('[data-action]');
  if (!btn) return;
  const id     = btn.closest('[data-id]').dataset.id;
  const action = btn.dataset.action;

  if (action === 'toggle')  TodoList.toggleTask(id);
  if (action === 'edit')    TodoList.enterEditMode(id);
  if (action === 'delete')  TodoList.deleteTask(id);
  if (action === 'confirm') TodoList.confirmEdit(id);
  if (action === 'cancel')  TodoList.cancelEdit(id);
});
```

The same delegation pattern is used for `#links-grid`.

### Pattern 3: Timer Button Listeners

```js
document.getElementById('btn-start').addEventListener('click', () => FocusTimer.start());
document.getElementById('btn-stop' ).addEventListener('click', () => FocusTimer.stop());
document.getElementById('btn-reset').addEventListener('click', () => FocusTimer.reset());
```

### Pattern 4: Storage Error Toast

```js
// StorageService.save() calls this on failure
function showStorageWarning(message) {
  const el = document.getElementById('storage-warning');
  el.textContent = message;
  el.classList.remove('hidden');
  setTimeout(() => el.classList.add('hidden'), 5000);
}
```

---

## Error Handling

| Scenario | Handling |
|---|---|
| `localStorage` unavailable on write | `StorageService.save()` catches the error, calls `showStorageWarning()`, returns `false`. In-memory state is preserved. |
| `localStorage` unavailable on read | `StorageService.load()` returns `null`; widget treats it as empty initial state. |
| Malformed JSON in `localStorage` | `JSON.parse` wrapped in try/catch; returns `null` on failure. Widget starts with empty state. |
| Empty/whitespace task text | Rejected at form submit handler; focus returned to input. |
| Empty label or URL for link | Rejected at form submit handler; focus returned to first empty field. |
| `crypto.randomUUID` unavailable | Fallback to `Date.now().toString() + Math.random().toString(36).slice(2)`. |

---

## Testing Strategy

### Unit Tests

Unit tests cover specific examples and edge cases for pure functions:

- `getGreeting(hour)` — all six boundary hours (5, 12, 18, 22, 0, 4)
- `formatTime(seconds)` — 1500 → "25:00", 0 → "00:00", 61 → "01:01"
- `StorageService.load()` — missing key returns null, malformed JSON returns null
- `StorageService.save()` — successful write returns true, unavailable storage returns false
- Task validation — empty string rejected, whitespace-only string rejected, valid text accepted
- Link validation — empty label rejected, empty URL rejected, both provided accepted

### Property-Based Tests

Property tests verify universal invariants across randomly generated inputs. The recommended library is **fast-check** (JavaScript), loaded via CDN for test runs, or any equivalent PBT library.

Each property test runs a minimum of **100 iterations**.

Tag format: `Feature: todo-life-dashboard, Property {N}: {property_text}`


---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

The properties below were derived from the acceptance criteria in `requirements.md`. Each is universally quantified and suitable for property-based testing using **fast-check** (or equivalent). Each test must run a minimum of **100 iterations**.

Tag format for test files: `Feature: todo-life-dashboard, Property {N}: {property_text}`

---

### Property 1: Greeting correctness

*For any* integer hour in [0, 23], `getGreeting(hour)` SHALL return "Good Morning" when hour ∈ [5, 12), "Good Afternoon" when hour ∈ [12, 18), "Good Evening" when hour ∈ [18, 22), and "Good Night" when hour ∈ [0, 5) ∪ [22, 24).

**Validates: Requirements 1.3, 1.4, 1.5, 1.6**

---

### Property 2: Time formatting produces valid HH:MM strings

*For any* integer hour in [0, 23] and integer minute in [0, 59], the time-formatting function SHALL return a string matching the pattern `HH:MM` where HH is the zero-padded hour and MM is the zero-padded minute.

**Validates: Requirements 1.1**

---

### Property 3: Timer tick decrements remaining by one per call

*For any* remaining value N in [1, 1500], after calling `FocusTimer.tick()` exactly K times (where K ≤ N), the remaining value SHALL equal N − K.

**Validates: Requirements 2.2**

---

### Property 4: Timer display always reflects current remaining time

*For any* remaining value in [0, 1500], calling `FocusTimer.render()` SHALL update the `#timer-display` element's text content to equal `formatTime(remaining)` (i.e., the MM:SS string representation of that value).

**Validates: Requirements 2.3, 11.5**

---

### Property 5: Timer stop retains remaining time

*For any* timer state with remaining value R and `isRunning === true`, calling `FocusTimer.stop()` SHALL leave remaining equal to R and set `isRunning` to false.

**Validates: Requirements 2.4**

---

### Property 6: Timer reset always restores to 1500

*For any* timer state (any remaining value, any running/stopped state), calling `FocusTimer.reset()` SHALL set remaining to 1500 and `isRunning` to false.

**Validates: Requirements 2.5**

---

### Property 7: Start button is disabled while timer is running

*For any* timer state, after calling `FocusTimer.start()`, the `#btn-start` element SHALL have the `disabled` attribute set; after calling `FocusTimer.stop()` or `FocusTimer.reset()`, the `#btn-start` element SHALL NOT have the `disabled` attribute.

**Validates: Requirements 2.7**

---

### Property 8: Adding a valid task grows the task list by one

*For any* task list of length N and any non-empty, non-whitespace string S, calling `TodoList.addTask(S)` SHALL result in a task list of length N + 1, where the last element has `text === S.trim()` and `completed === false`.

**Validates: Requirements 3.2**

---

### Property 9: Whitespace-only and empty inputs are rejected for tasks and links

*For any* string composed entirely of whitespace characters (spaces, tabs, newlines), calling `TodoList.addTask(s)` SHALL leave the task list unchanged; calling `TodoList.editTask(id, s)` SHALL leave the task's text unchanged; calling `QuickLinks.addLink(s, url)` or `QuickLinks.addLink(label, s)` with a whitespace-only field SHALL leave the links list unchanged.

**Validates: Requirements 3.3, 4.4, 8.3**

---

### Property 10: Task insertion order is preserved in the rendered list

*For any* sequence of N valid task additions, the rendered `#task-list` SHALL display tasks in the same order they were added (first-in, first-displayed).

**Validates: Requirements 3.4**

---

### Property 11: Task localStorage round-trip preserves all task data

*For any* array of Task objects written to `localStorage` under key `todo_life_tasks`, calling `TodoList.init()` (which reads from localStorage) SHALL produce an in-memory `tasks` array that is deeply equal to the original array (same ids, texts, completed statuses, and createdAt values).

**Validates: Requirements 3.5, 6.2**

---

### Property 12: Each rendered task item contains all required controls

*For any* task list of length N ≥ 1, every rendered `<li>` element in `#task-list` SHALL contain exactly one element with `data-action="toggle"`, one with `data-action="edit"`, and one with `data-action="delete"`.

**Validates: Requirements 4.1, 5.1, 5.4**

---

### Property 13: Edit mode pre-fills input with current task text

*For any* task with text T, calling `TodoList.enterEditMode(id)` SHALL render an `<input>` element whose `value` equals T.

**Validates: Requirements 4.2**

---

### Property 14: Confirming a valid edit updates the task text

*For any* task with original text T and any non-empty, non-whitespace replacement text T′, calling `TodoList.confirmEdit(id)` with value T′ SHALL update the task's `text` to `T′.trim()` and persist the change to localStorage.

**Validates: Requirements 4.3, 4.5**

---

### Property 15: Completion toggle is an involution

*For any* task with completion status B, calling `TodoList.toggleTask(id)` twice SHALL return the task to completion status B (i.e., toggle is its own inverse). After the first toggle, the status SHALL be `!B`; after the second, it SHALL be `B` again.

**Validates: Requirements 5.2, 5.3**

---

### Property 16: Any task mutation is immediately persisted to localStorage

*For any* task collection and any single mutation (add, edit, toggle, or delete), after the mutation completes, `JSON.parse(localStorage.getItem('todo_life_tasks'))` SHALL deeply equal the current in-memory `tasks` array.

**Validates: Requirements 5.6, 6.1**

---

### Property 17: Link localStorage round-trip preserves all link data

*For any* array of Link objects written to `localStorage` under key `todo_life_links`, calling `QuickLinks.init()` SHALL produce an in-memory `links` array that is deeply equal to the original array (same ids, labels, and URLs).

**Validates: Requirements 7.3**

---

### Property 18: Link buttons display correct labels and open correct URLs

*For any* array of Link objects, after `QuickLinks.render()`, each link SHALL have a corresponding button whose visible text equals `link.label`, and clicking that button SHALL invoke `window.open` with `link.url` and `'_blank'` as arguments.

**Validates: Requirements 7.1, 7.2**

---

### Property 19: Adding a valid link grows the links list by one

*For any* links list of length N and any non-empty label L and non-empty URL U, calling `QuickLinks.addLink(L, U)` SHALL result in a links list of length N + 1, where the last element has `label === L.trim()` and `url === U.trim()`, and the updated list is persisted to localStorage.

**Validates: Requirements 8.2**

---

### Property 20: Deleting a task or link removes it and persists the change

*For any* collection of length N ≥ 1 and any element E in that collection, calling the corresponding delete function SHALL result in a collection of length N − 1 that does not contain E, and the updated collection SHALL be persisted to localStorage.

**Validates: Requirements 5.5, 8.5**

---

### Property 21: JSON serialization round-trip is lossless for all data types

*For any* valid `Task[]` or `Link[]` array, `JSON.parse(JSON.stringify(collection))` SHALL produce an array that is deeply equal to the original (same length, same field values for all elements).

**Validates: Requirements 9.3, 9.4**

---

## Error Handling

| Scenario | Handling |
|---|---|
| `localStorage` unavailable on write | `StorageService.save()` catches the error, calls `showStorageWarning()`, returns `false`. In-memory state is preserved. |
| `localStorage` unavailable on read | `StorageService.load()` returns `null`; widget treats it as empty initial state. |
| Malformed JSON in `localStorage` | `JSON.parse` wrapped in try/catch; returns `null` on failure. Widget starts with empty state. |
| Empty/whitespace task text | Rejected at form submit handler; focus returned to input. |
| Empty label or URL for link | Rejected at form submit handler; focus returned to first empty field. |
| `crypto.randomUUID` unavailable | Fallback to `Date.now().toString() + Math.random().toString(36).slice(2)`. |

---

## Testing Strategy

### Dual Testing Approach

Both unit tests and property-based tests are used together for comprehensive coverage:

- **Unit tests** — specific examples, edge cases, and error conditions
- **Property tests** — universal invariants across randomly generated inputs (Properties 1–21 above)

### Unit Test Coverage

Unit tests focus on concrete examples and boundary conditions:

- `getGreeting(hour)` — boundary hours: 5, 11, 12, 17, 18, 21, 22, 0, 4
- `formatTime(seconds)` — 1500 → "25:00", 0 → "00:00", 61 → "01:01", 599 → "09:59"
- `StorageService.load()` — missing key returns null, malformed JSON returns null
- `StorageService.save()` — successful write returns true, unavailable storage returns false and shows warning
- Timer edge case: remaining reaches 0 → `isRunning === false`, `.timer-finished` class applied
- Empty localStorage on init → tasks/links render as empty with no errors

### Property-Based Test Configuration

- **Library**: [fast-check](https://github.com/dubzzz/fast-check) (JavaScript), loaded via CDN for test runs
- **Minimum iterations**: 100 per property test
- **Tag format**: `Feature: todo-life-dashboard, Property {N}: {property_text}`
- Each correctness property (1–21) maps to exactly one property-based test

### Test File Structure

```
tests/
  unit/
    greeting.test.js     — Properties 1, 2; unit boundary tests
    timer.test.js        — Properties 3–7; unit edge cases
    todo.test.js         — Properties 8–16; unit edge cases
    links.test.js        — Properties 17–20; unit edge cases
    storage.test.js      — Property 21; StorageService unit tests
  property/
    greeting.prop.js     — Properties 1, 2
    timer.prop.js        — Properties 3–7
    todo.prop.js         — Properties 8–16
    links.prop.js        — Properties 17–20
    storage.prop.js      — Property 21
```

### Why PBT Applies Here

The core logic of this application — greeting classification, time formatting, task/link CRUD, localStorage serialization — consists of pure or near-pure functions whose correctness should hold across a wide input space. Property-based testing is well-suited because:

- Input variation (different strings, hours, task collections) reveals edge cases that example tests miss
- The functions are cheap to execute (no network, no heavy I/O), making 100+ iterations cost-effective
- Several requirements are explicitly stated as universal properties (e.g., Requirement 9.4)
