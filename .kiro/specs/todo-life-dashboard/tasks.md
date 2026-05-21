# Implementation Plan: Todo-Life Dashboard

## Overview

This plan implements the Todo-Life Dashboard — a self-contained, client-side productivity app delivered as three files (`index.html`, `css/style.css`, `js/app.js`) with no build step or external dependencies. Implementation is organised into six phases: project scaffold, core widget logic, CSS styling, integration/accessibility, unit tests, and property-based tests.

## Task Dependency Graph

```json
{
  "waves": [
    {
      "wave": 1,
      "tasks": ["1"],
      "description": "Project scaffold — create all files and implement CONFIG + StorageService"
    },
    {
      "wave": 2,
      "tasks": ["2", "3"],
      "description": "Core widget logic (js/app.js) and CSS styling (css/style.css) — can proceed in parallel after scaffold"
    },
    {
      "wave": 3,
      "tasks": ["4"],
      "description": "Integration, polish, and accessibility — requires all widgets and styles to be complete"
    },
    {
      "wave": 4,
      "tasks": ["5", "6"],
      "description": "Unit tests and property-based tests — requires all implementation to be complete"
    }
  ]
}
```

## Tasks

- [ ] 1. Set up project scaffold
  - Create the directory structure: project root with `css/` and `js/` subdirectories, and `tests/unit/` and `tests/property/` subdirectories.
  - No widget logic yet — just the file skeletons, CONFIG, and StorageService.

  - [x] 1.1 Create project file structure
    - Create the following empty (or minimal) files: `index.html`, `css/style.css`, `js/app.js`, `tests/unit/greeting.test.js`, `tests/unit/timer.test.js`, `tests/unit/todo.test.js`, `tests/unit/links.test.js`, `tests/unit/storage.test.js`, `tests/property/greeting.prop.js`, `tests/property/timer.prop.js`, `tests/property/todo.prop.js`, `tests/property/links.prop.js`, `tests/property/storage.prop.js`.
    - Acceptance: all files exist; no errors when opening `index.html` in a browser.

  - [x] 1.2 Create index.html shell
    - Implement the full HTML structure as specified in the design document's "HTML Layout" section.
    - Include: `<!DOCTYPE html>`, `<meta charset>`, `<meta viewport>`, `<title>Todo-Life Dashboard</title>`, link to `css/style.css`, `<script defer src="js/app.js">`.
    - Include all four widget `<section>` elements with their correct `id`, `class`, and `aria-label` attributes.
    - Include the `#storage-warning` div with `role="alert"` and `aria-live="polite"`.
    - Acceptance: HTML validates; all required IDs are present (`#greeting-widget`, `#focus-timer`, `#todo-list`, `#quick-links`, `#timer-display`, `#btn-start`, `#btn-stop`, `#btn-reset`, `#todo-form`, `#todo-input`, `#task-list`, `#links-form`, `#link-label-input`, `#link-url-input`, `#links-grid`, `#storage-warning`).

  - [x] 1.3 Create css/style.css skeleton with design tokens
    - Create `css/style.css` with the 12 commented sections as listed in the design document's "CSS Architecture" section.
    - Populate only the CSS Custom Properties section (design tokens) with all variables from the design document.
    - Leave all other sections as empty comment blocks for now.
    - Acceptance: `css/style.css` loads without errors; all CSS custom properties are defined on `:root`.

  - [-] 1.4 Create js/app.js skeleton with CONFIG and StorageService
    - Create `js/app.js` with the top-level structure: `CONFIG` constant, `StorageService` object, empty `GreetingWidget`, `FocusTimer`, `TodoList`, `QuickLinks` objects, and an `init()` function called on `DOMContentLoaded`.
    - Implement `CONFIG` with all constants: `TIMER_DURATION_SECONDS`, `STORAGE_KEY_TASKS`, `STORAGE_KEY_LINKS`, `GREETING_THRESHOLDS`.
    - Implement `StorageService.load(key)`, `StorageService.save(key, value)`, and `StorageService.isAvailable()` with full try/catch error handling as specified in the design document.
    - Implement the `showStorageWarning(message)` helper function.
    - Implement the `crypto.randomUUID` fallback for ID generation.
    - Acceptance: `app.js` loads without syntax errors; `StorageService.load('nonexistent')` returns `null`; `StorageService.save` and `StorageService.load` round-trip a simple object correctly.

- [ ] 2. Implement core widget logic in js/app.js
  - Implement all four widget modules with their full state, render, event handler, and init logic.

  - [~] 2.1 Implement GreetingWidget
    - Implement `getGreeting(hour)` pure function returning the correct greeting string for any hour 0–23, using the thresholds in `CONFIG.GREETING_THRESHOLDS`.
    - Implement `formatTime(date)` using `Date#toLocaleTimeString` with `{ hour: '2-digit', minute: '2-digit', hour12: false }`.
    - Implement `formatDate(date)` using `Date#toLocaleDateString` with `{ weekday: 'long', day: 'numeric', month: 'long', year: 'numeric' }`.
    - Implement `GreetingWidget.tick()` to read `new Date()`, call `getGreeting`, `formatTime`, `formatDate`, and update `#greeting-text`, `#time-display`, `#date-display`.
    - Implement `GreetingWidget.init()` to call `tick()` immediately and then start a `setInterval(tick, 60_000)`.
    - Acceptance: Opening the page shows the correct greeting, time, and date; the time updates every minute.

  - [~] 2.2 Implement FocusTimer
    - Implement `timerState` object with `remaining`, `intervalId`, and `isRunning` fields.
    - Implement `formatTime(seconds)` pure function returning `MM:SS` string (e.g., 1500 → "25:00", 61 → "01:01", 0 → "00:00").
    - Implement `FocusTimer.render()` to write `formatTime(timerState.remaining)` to `#timer-display`.
    - Implement `FocusTimer.tick()` to decrement `remaining` by 1, call `render()`, and when `remaining` reaches 0: clear the interval, set `isRunning = false`, add `.timer-finished` class to `#focus-timer`, and disable `#btn-start`.
    - Implement `FocusTimer.start()` to set `isRunning = true`, disable `#btn-start`, and start the interval calling `tick()` every 1000ms.
    - Implement `FocusTimer.stop()` to clear the interval, set `isRunning = false`, and enable `#btn-start`.
    - Implement `FocusTimer.reset()` to call `stop()`, restore `remaining` to `CONFIG.TIMER_DURATION_SECONDS`, remove `.timer-finished` class, and call `render()`.
    - Implement `FocusTimer.init()` to attach click listeners to `#btn-start`, `#btn-stop`, `#btn-reset` and call `render()`.
    - Acceptance: Timer starts at 25:00; start/stop/reset controls work correctly; start button is disabled while running; `.timer-finished` class is applied when countdown reaches 00:00.

  - [~] 2.3 Implement TodoList
    - Implement `tasks` array state.
    - Implement `TodoList.render()` to clear `#task-list` and rebuild it from `tasks[]`. Each `<li>` must have `data-id` attribute and contain: a toggle button with `data-action="toggle"`, a text span, an edit button with `data-action="edit"`, and a delete button with `data-action="delete"`. Apply `.task-item--completed` class to completed tasks.
    - Implement `TodoList.addTask(text)` to validate (reject empty/whitespace), create a Task object with `id`, `text` (trimmed), `completed: false`, `createdAt: Date.now()`, push to `tasks`, call `save()` and `render()`.
    - Implement `TodoList.toggleTask(id)` to flip `completed`, call `save()` and `render()`.
    - Implement `TodoList.deleteTask(id)` to filter out the task, call `save()` and `render()`.
    - Implement `TodoList.enterEditMode(id)` to replace the task `<li>` with an inline `<input>` pre-filled with the task's current text, plus confirm (`data-action="confirm"`) and cancel (`data-action="cancel"`) buttons. Only one task can be in edit mode at a time.
    - Implement `TodoList.confirmEdit(id)` to validate the new text (reject empty/whitespace), update `task.text`, call `save()` and `render()`.
    - Implement `TodoList.cancelEdit(id)` to call `render()` without saving.
    - Implement `TodoList.save()` to call `StorageService.save(CONFIG.STORAGE_KEY_TASKS, tasks)`.
    - Implement `TodoList.init()` to load from storage, initialise `tasks` (default to `[]` if null), attach the `submit` listener on `#todo-form`, attach the delegated click listener on `#task-list`, and call `render()`.
    - Acceptance: Tasks can be added, edited, toggled, and deleted; all changes persist across page reloads; empty/whitespace inputs are rejected; edit mode pre-fills the input with current text.

  - [~] 2.4 Implement QuickLinks
    - Implement `links` array state.
    - Implement `QuickLinks.render()` to clear `#links-grid` and rebuild it from `links[]`. Each link renders as a button with visible text equal to `link.label` and a delete button with `data-action="delete"`. Clicking the link button calls `window.open(link.url, '_blank')`.
    - Implement `QuickLinks.addLink(label, url)` to validate both fields (reject empty/whitespace), create a Link object with `id`, `label` (trimmed), `url` (trimmed), push to `links`, call `save()` and `render()`.
    - Implement `QuickLinks.deleteLink(id)` to filter out the link, call `save()` and `render()`.
    - Implement `QuickLinks.save()` to call `StorageService.save(CONFIG.STORAGE_KEY_LINKS, links)`.
    - Implement `QuickLinks.init()` to load from storage, initialise `links` (default to `[]` if null), attach the `submit` listener on `#links-form`, attach the delegated click listener on `#links-grid`, and call `render()`.
    - Acceptance: Links can be added and deleted; clicking a link opens the URL in a new tab; all changes persist across page reloads; empty label or URL is rejected with focus returned to the first empty field.

- [ ] 3. Implement CSS styling in css/style.css
  - Fill in all 12 sections of the stylesheet.

  - [~] 3.1 Implement CSS reset and base styles
    - Add a minimal CSS reset (box-sizing, margin/padding reset, `inherit` font).
    - Set `font-family` to `var(--font-family)` on `body`, `background-color` to `var(--color-bg)`, and `color` to `var(--color-text-primary)`.
    - Acceptance: Page renders without browser default margins/padding artifacts.

  - [~] 3.2 Implement dashboard grid layout and widget card styles
    - Implement `.dashboard` with CSS Grid, `gap: var(--space-6)`, `padding: var(--space-6)`, `max-width: 1200px`, `margin: 0 auto`, and `grid-template-columns: 1fr` (mobile-first).
    - Implement `.widget` card styles: `background: var(--color-surface)`, `border: 1px solid var(--color-border)`, `border-radius: var(--radius-lg)`, `padding: var(--space-6)`, `box-shadow: var(--shadow-card)`.
    - Implement `.widget__title` styles: `font-size: var(--font-size-lg)`, `font-weight: 600`, `margin-bottom: var(--space-4)`.
    - Acceptance: Widgets appear as distinct cards on a light background.

  - [~] 3.3 Implement widget-specific styles
    - **Greeting Widget**: Style `#time-display` (`font-size: var(--font-size-xl)`, `font-weight: 700`) and `#date-display` (`color: var(--color-text-secondary)`).
    - **Focus Timer**: Style `#timer-display` (`font-size: var(--font-size-2xl)`, `font-weight: 700`, `font-variant-numeric: tabular-nums`, centered). Style `.timer__controls` with flex layout. Style `.timer-finished` background (`var(--color-finished)`).
    - **To-Do List**: Style `.task-list` (remove list bullets). Style `.task-item` with flex layout. Style `.task-item--completed .task-item__text` with `text-decoration: line-through` and `color: var(--color-text-secondary)`.
    - **Quick Links**: Style `.links__grid` with CSS Grid or flex wrap. Style link buttons to be visually distinct from action buttons.
    - **Buttons**: Implement `.btn`, `.btn--primary` (accent colour), `.btn--secondary`, `.btn--ghost`, `.btn--danger` with hover states.
    - **Form inputs**: Style `.input` with border, border-radius, padding, and focus ring.
    - **Utility classes**: Implement `.hidden` (`display: none`) and `.sr-only` (visually hidden but accessible).
    - **Storage warning**: Style `#storage-warning` as a non-blocking banner.
    - Acceptance: All widgets are visually styled; completed tasks show strikethrough; timer finished state shows green background; buttons have clear hover states.

  - [~] 3.4 Implement responsive breakpoints
    - At `min-width: 640px`: set `.dashboard` to `grid-template-columns: repeat(2, 1fr)`.
    - At `min-width: 1024px`: set `.dashboard` to `grid-template-columns: repeat(4, 1fr)`; set `.widget--greeting` and `.widget--todo` to `grid-column: span 2`.
    - Acceptance: Layout is single-column on mobile, two-column at 640px, four-column at 1024px with Greeting and Todo spanning two columns each.

- [ ] 4. Integration, polish, and accessibility
  - Wire everything together and verify the full application works end-to-end.

  - [~] 4.1 Wire init() bootstrap and verify all widgets load
    - Implement the `init()` function to call `GreetingWidget.init()`, `FocusTimer.init()`, `TodoList.init()`, and `QuickLinks.init()` on `DOMContentLoaded`.
    - Open `index.html` in a browser and verify: greeting/time/date display correctly, timer starts at 25:00, task list loads from localStorage (or renders empty), quick links load from localStorage (or render empty).
    - Acceptance: All four widgets initialise without console errors; the page is interactive within 2 seconds.

  - [~] 4.2 Implement and verify storage error toast UI
    - Verify `showStorageWarning(message)` correctly shows `#storage-warning`, sets its text, and hides it after 5 seconds.
    - Test by temporarily making `localStorage` unavailable and confirming the warning appears on a save attempt.
    - Acceptance: Warning banner appears and auto-dismisses; in-memory state is preserved even when storage fails.

  - [~] 4.3 Accessibility audit and ARIA attributes
    - Verify all interactive elements have accessible labels (buttons, inputs).
    - Verify `#timer-display` has `aria-live="polite"` and `aria-atomic="true"`.
    - Verify `#storage-warning` has `role="alert"` and `aria-live="polite"`.
    - Verify all four widget `<section>` elements have `aria-label` attributes.
    - Verify keyboard navigation works for all controls (Tab, Enter, Space).
    - Verify colour contrast meets WCAG 2.1 AA (≥ 4.5:1 for normal text, ≥ 3:1 for large text) for all text/background pairs listed in the design document.
    - Acceptance: No accessibility violations for the attributes and contrast ratios specified in the design document.

- [ ] 5. Write unit tests
  - Implement unit tests for all pure functions and widget logic.

  - [ ] 5.1 Set up test infrastructure
    - Create a minimal test runner HTML file (`tests/test-runner.html`) that loads fast-check via CDN and a simple assertion helper, or configure a Node.js test runner (e.g., Jest or Vitest) with a `package.json`.
    - Ensure all test files can be run and results are visible.
    - Acceptance: Running the test suite produces a pass/fail report; at least one trivial test passes.

  - [ ] 5.2 Write unit tests for GreetingWidget (tests/unit/greeting.test.js)
    - Test `getGreeting(hour)` at all boundary hours: 0, 4, 5, 11, 12, 17, 18, 21, 22, 23.
    - Test `formatTime` output format: verify "25:00" for 1500s, "00:00" for 0s, "01:01" for 61s, "09:59" for 599s.
    - Acceptance: All unit tests pass.

  - [ ] 5.3 Write unit tests for FocusTimer (tests/unit/timer.test.js)
    - Test `formatTime(seconds)` with boundary values: 0, 1, 59, 60, 61, 1499, 1500.
    - Test that `FocusTimer.reset()` restores `remaining` to 1500 and `isRunning` to false from any state.
    - Test that when `remaining` reaches 0 after `tick()`, `isRunning` is false and `.timer-finished` class is applied.
    - Acceptance: All unit tests pass.

  - [ ] 5.4 Write unit tests for TodoList (tests/unit/todo.test.js)
    - Test `addTask` with a valid string: task list grows by 1, new task has correct `text` (trimmed), `completed: false`.
    - Test `addTask` with empty string and whitespace-only string: task list unchanged in both cases.
    - Test `toggleTask`: flipping twice returns to original state.
    - Test `deleteTask`: list shrinks by 1 and the deleted task is absent.
    - Test `confirmEdit` with valid text: task text is updated. With empty/whitespace text: task text is unchanged.
    - Test `StorageService.load` with missing key: returns null. With malformed JSON: returns null.
    - Acceptance: All unit tests pass.

  - [ ] 5.5 Write unit tests for QuickLinks and StorageService (tests/unit/links.test.js, tests/unit/storage.test.js)
    - Test `addLink` with valid label and URL: links list grows by 1, new link has correct `label` and `url` (trimmed).
    - Test `addLink` with empty label and with empty URL: links list unchanged in both cases.
    - Test `deleteLink`: list shrinks by 1 and the deleted link is absent.
    - Test `StorageService.save` + `StorageService.load` round-trip: saved object equals loaded object. `save` returns `true` on success.
    - Acceptance: All unit tests pass.

- [ ] 6. Write property-based tests
  - Implement all 21 correctness properties from the design document using fast-check (minimum 100 iterations each).

  - [ ] 6.1 Write property tests for GreetingWidget (tests/property/greeting.prop.js)
    - **Property 1**: For any integer hour in [0, 23], `getGreeting(hour)` returns the correct greeting string.
      Tag: `Feature: todo-life-dashboard, Property 1: Greeting correctness`
      **Validates: Requirements 1.3, 1.4, 1.5, 1.6**
    - **Property 2**: For any integer hour in [0, 23] and minute in [0, 59], the time-formatting function returns a string matching `HH:MM`.
      Tag: `Feature: todo-life-dashboard, Property 2: Time formatting produces valid HH:MM strings`
      **Validates: Requirements 1.1**
    - Acceptance: Both properties pass 100+ iterations without counterexamples.

  - [ ] 6.2 Write property tests for FocusTimer (tests/property/timer.prop.js)
    - **Property 3**: For any remaining N in [1, 1500], after K ticks (K ≤ N), remaining equals N − K.
      Tag: `Feature: todo-life-dashboard, Property 3: Timer tick decrements remaining by one per call`
      **Validates: Requirements 2.2**
    - **Property 4**: For any remaining in [0, 1500], `FocusTimer.render()` sets `#timer-display` text to `formatTime(remaining)`.
      Tag: `Feature: todo-life-dashboard, Property 4: Timer display always reflects current remaining time`
      **Validates: Requirements 2.3, 11.5**
    - **Property 5**: For any running timer state with remaining R, `FocusTimer.stop()` leaves remaining equal to R and sets `isRunning` to false.
      Tag: `Feature: todo-life-dashboard, Property 5: Timer stop retains remaining time`
      **Validates: Requirements 2.4**
    - **Property 6**: For any timer state, `FocusTimer.reset()` sets remaining to 1500 and `isRunning` to false.
      Tag: `Feature: todo-life-dashboard, Property 6: Timer reset always restores to 1500`
      **Validates: Requirements 2.5**
    - **Property 7**: After `FocusTimer.start()`, `#btn-start` has `disabled`; after `stop()` or `reset()`, it does not.
      Tag: `Feature: todo-life-dashboard, Property 7: Start button is disabled while timer is running`
      **Validates: Requirements 2.7**
    - Acceptance: All five properties pass 100+ iterations without counterexamples.

  - [ ] 6.3 Write property tests for TodoList (tests/property/todo.prop.js)
    - **Property 8**: For any task list of length N and any non-empty string S, `addTask(S)` results in length N + 1 with last element `text === S.trim()` and `completed === false`.
      Tag: `Feature: todo-life-dashboard, Property 8: Adding a valid task grows the task list by one`
      **Validates: Requirements 3.2**
    - **Property 9**: For any whitespace-only string, `addTask` leaves the list unchanged; `editTask` with whitespace leaves text unchanged; `addLink` with a whitespace field leaves links unchanged.
      Tag: `Feature: todo-life-dashboard, Property 9: Whitespace-only and empty inputs are rejected for tasks and links`
      **Validates: Requirements 3.3, 4.4, 8.3**
    - **Property 10**: For any sequence of N valid task additions, `#task-list` displays tasks in insertion order.
      Tag: `Feature: todo-life-dashboard, Property 10: Task insertion order is preserved in the rendered list`
      **Validates: Requirements 3.4**
    - **Property 11**: For any Task[] written to localStorage, `TodoList.init()` produces an in-memory array deeply equal to the original.
      Tag: `Feature: todo-life-dashboard, Property 11: Task localStorage round-trip preserves all task data`
      **Validates: Requirements 3.5, 6.2**
    - **Property 12**: For any task list of length N ≥ 1, every rendered `<li>` contains exactly one toggle, one edit, and one delete control.
      Tag: `Feature: todo-life-dashboard, Property 12: Each rendered task item contains all required controls`
      **Validates: Requirements 4.1, 5.1, 5.4**
    - **Property 13**: For any task with text T, `enterEditMode(id)` renders an `<input>` whose value equals T.
      Tag: `Feature: todo-life-dashboard, Property 13: Edit mode pre-fills input with current task text`
      **Validates: Requirements 4.2**
    - **Property 14**: For any task with text T and any non-empty replacement T′, `confirmEdit(id)` updates text to `T′.trim()` and persists to localStorage.
      Tag: `Feature: todo-life-dashboard, Property 14: Confirming a valid edit updates the task text`
      **Validates: Requirements 4.3, 4.5**
    - **Property 15**: For any task with completion status B, toggling twice returns to status B.
      Tag: `Feature: todo-life-dashboard, Property 15: Completion toggle is an involution`
      **Validates: Requirements 5.2, 5.3**
    - **Property 16**: After any single mutation (add, edit, toggle, delete), `JSON.parse(localStorage.getItem('todo_life_tasks'))` deeply equals the in-memory `tasks` array.
      Tag: `Feature: todo-life-dashboard, Property 16: Any task mutation is immediately persisted to localStorage`
      **Validates: Requirements 5.6, 6.1**
    - Acceptance: All nine properties pass 100+ iterations without counterexamples.

  - [ ] 6.4 Write property tests for QuickLinks (tests/property/links.prop.js)
    - **Property 17**: For any Link[] written to localStorage, `QuickLinks.init()` produces an in-memory array deeply equal to the original.
      Tag: `Feature: todo-life-dashboard, Property 17: Link localStorage round-trip preserves all link data`
      **Validates: Requirements 7.3**
    - **Property 18**: For any Link[], after `render()`, each link has a button with visible text equal to `link.label`, and clicking it calls `window.open(link.url, '_blank')`.
      Tag: `Feature: todo-life-dashboard, Property 18: Link buttons display correct labels and open correct URLs`
      **Validates: Requirements 7.1, 7.2**
    - **Property 19**: For any links list of length N and non-empty L and U, `addLink(L, U)` results in length N + 1 with `label === L.trim()` and `url === U.trim()`, persisted to localStorage.
      Tag: `Feature: todo-life-dashboard, Property 19: Adding a valid link grows the links list by one`
      **Validates: Requirements 8.2**
    - **Property 20**: For any collection of length N ≥ 1 and any element E, the delete function results in length N − 1 without E, persisted to localStorage.
      Tag: `Feature: todo-life-dashboard, Property 20: Deleting a task or link removes it and persists the change`
      **Validates: Requirements 5.5, 8.5**
    - Acceptance: All four properties pass 100+ iterations without counterexamples.

  - [ ] 6.5 Write property tests for storage round-trip (tests/property/storage.prop.js)
    - **Property 21**: For any valid `Task[]` or `Link[]`, `JSON.parse(JSON.stringify(collection))` produces an array deeply equal to the original.
      Tag: `Feature: todo-life-dashboard, Property 21: JSON serialization round-trip is lossless for all data types`
      **Validates: Requirements 9.3, 9.4**
    - Acceptance: Property passes 100+ iterations without counterexamples.

## Notes

- All implementation is constrained to three files: `index.html`, `css/style.css`, `js/app.js`. No frameworks, build tools, or external dependencies are used at runtime.
- Property-based tests use [fast-check](https://github.com/dubzzz/fast-check) loaded via CDN. Each property test runs a minimum of 100 iterations.
- Waves 2 and 3 (widget logic and CSS) can be worked on in parallel since they touch different files.
- The `formatTime(seconds)` function in `FocusTimer` is distinct from the `formatTime(date)` function in `GreetingWidget` — name them clearly in `app.js` to avoid collision (e.g., `formatTimerDisplay` vs. `formatClockTime`).
- `crypto.randomUUID` is available in all modern browsers (Chrome 92+, Firefox 95+, Safari 15.4+) but a fallback using `Date.now() + Math.random()` must be implemented per the design document.
- The storage error toast auto-dismisses after 5 seconds; in-memory state is always preserved regardless of storage failures.
