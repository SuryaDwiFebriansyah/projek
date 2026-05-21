# Requirements Document

## Introduction

The Todo-Life Dashboard is a client-side web application that serves as a personal productivity hub. It combines a live greeting with time/date display, a Pomodoro-style focus timer, a persistent to-do list, and a quick-links panel — all in a single, self-contained HTML page. No backend server is required; all data is stored in the browser's Local Storage. The app can be used as a standalone web page or packaged as a browser extension.

The implementation is constrained to a single HTML file, one CSS file (`css/style.css`), and one JavaScript file (`js/app.js`). No frameworks, build tools, or test infrastructure are required.

---

## Glossary

- **Dashboard**: The single-page web application described in this document.
- **Greeting_Widget**: The UI component that displays the current time, date, and a time-of-day greeting message.
- **Focus_Timer**: The UI component that implements a 25-minute countdown timer with start, stop, and reset controls.
- **Todo_List**: The UI component that manages a collection of user-defined tasks.
- **Task**: A single item in the Todo_List, consisting of a text description and a completion status.
- **Quick_Links**: The UI component that displays a set of user-defined shortcut buttons, each opening a URL in a new browser tab.
- **Link**: A single item in the Quick_Links panel, consisting of a label and a URL.
- **Local_Storage**: The browser's `localStorage` API used to persist all user data client-side.
- **Modern_Browser**: Chrome, Firefox, Edge, or Safari at their current stable release at the time of deployment.

---

## Requirements

### Requirement 1: Live Greeting and Date/Time Display

**User Story:** As a user, I want to see the current time, date, and a contextual greeting when I open the Dashboard, so that I have an immediate sense of the time of day without switching apps.

#### Acceptance Criteria

1. THE Greeting_Widget SHALL display the current time in HH:MM format, updated every minute.
2. THE Greeting_Widget SHALL display the current date in a human-readable format (e.g., "Monday, 26 May 2025").
3. WHEN the local hour is between 05:00 and 11:59, THE Greeting_Widget SHALL display the greeting "Good Morning".
4. WHEN the local hour is between 12:00 and 17:59, THE Greeting_Widget SHALL display the greeting "Good Afternoon".
5. WHEN the local hour is between 18:00 and 21:59, THE Greeting_Widget SHALL display the greeting "Good Evening".
6. WHEN the local hour is between 22:00 and 04:59, THE Greeting_Widget SHALL display the greeting "Good Night".

---

### Requirement 2: Focus Timer

**User Story:** As a user, I want a 25-minute countdown timer with start, stop, and reset controls, so that I can use the Pomodoro technique to manage focused work sessions.

#### Acceptance Criteria

1. THE Focus_Timer SHALL initialise with a countdown value of 25 minutes and 00 seconds (25:00).
2. WHEN the user activates the start control, THE Focus_Timer SHALL begin counting down one second per real-world second.
3. WHILE the Focus_Timer is counting down, THE Focus_Timer SHALL update the displayed time every second.
4. WHEN the user activates the stop control, THE Focus_Timer SHALL pause the countdown and retain the current remaining time.
5. WHEN the user activates the reset control, THE Focus_Timer SHALL stop any active countdown and restore the displayed time to 25:00.
6. WHEN the countdown reaches 00:00, THE Focus_Timer SHALL stop automatically and display a visual indication that the session has ended.
7. WHILE the Focus_Timer is counting down, THE Focus_Timer SHALL disable the start control to prevent duplicate timers.

---

### Requirement 3: To-Do List — Add and Display Tasks

**User Story:** As a user, I want to add tasks to a list and see them displayed, so that I can track what I need to accomplish.

#### Acceptance Criteria

1. THE Todo_List SHALL provide an input field and a submit control for entering new task text.
2. WHEN the user submits a non-empty task text, THE Todo_List SHALL append a new Task with the provided text and a default completion status of incomplete.
3. IF the user submits an empty or whitespace-only input, THEN THE Todo_List SHALL reject the submission and retain focus on the input field.
4. THE Todo_List SHALL display all Tasks in the order they were added.
5. WHEN the Dashboard is loaded, THE Todo_List SHALL restore all Tasks previously saved in Local_Storage.

---

### Requirement 4: To-Do List — Edit Tasks

**User Story:** As a user, I want to edit the text of an existing task, so that I can correct mistakes or update task descriptions without deleting and re-adding items.

#### Acceptance Criteria

1. THE Todo_List SHALL provide an edit control for each Task.
2. WHEN the user activates the edit control for a Task, THE Todo_List SHALL replace the Task's text display with an editable input field pre-filled with the current task text.
3. WHEN the user confirms the edit with a non-empty value, THE Todo_List SHALL update the Task's text to the new value and return to display mode.
4. IF the user confirms the edit with an empty or whitespace-only value, THEN THE Todo_List SHALL reject the change and retain the original task text.
5. WHEN a Task is edited, THE Todo_List SHALL persist the updated Task to Local_Storage.

---

### Requirement 5: To-Do List — Complete and Delete Tasks

**User Story:** As a user, I want to mark tasks as done and delete tasks I no longer need, so that I can maintain an accurate and uncluttered list.

#### Acceptance Criteria

1. THE Todo_List SHALL provide a completion toggle control for each Task.
2. WHEN the user activates the completion toggle for an incomplete Task, THE Todo_List SHALL mark the Task as complete and apply a distinct visual style (e.g., strikethrough text).
3. WHEN the user activates the completion toggle for a complete Task, THE Todo_List SHALL mark the Task as incomplete and remove the completed visual style.
4. THE Todo_List SHALL provide a delete control for each Task.
5. WHEN the user activates the delete control for a Task, THE Todo_List SHALL remove the Task from the list permanently.
6. WHEN a Task's completion status is changed or a Task is deleted, THE Todo_List SHALL persist the updated Task collection to Local_Storage.

---

### Requirement 6: To-Do List — Local Storage Persistence

**User Story:** As a user, I want my tasks to be saved automatically, so that my list is preserved across browser sessions without any manual export or login.

#### Acceptance Criteria

1. WHEN any Task is added, edited, completed, or deleted, THE Todo_List SHALL serialise the full Task collection and write it to Local_Storage under a fixed key.
2. WHEN the Dashboard is loaded, THE Todo_List SHALL read the Task collection from Local_Storage and render all stored Tasks.
3. IF no Task data exists in Local_Storage, THEN THE Todo_List SHALL render an empty list with no errors.

---

### Requirement 7: Quick Links — Display and Open

**User Story:** As a user, I want to see my saved favourite website shortcuts and open them with a single click, so that I can navigate to frequently visited sites quickly.

#### Acceptance Criteria

1. THE Quick_Links SHALL display each saved Link as a labelled button.
2. WHEN the user activates a Link button, THE Quick_Links SHALL open the associated URL in a new browser tab.
3. WHEN the Dashboard is loaded, THE Quick_Links SHALL restore all Links previously saved in Local_Storage.
4. IF no Link data exists in Local_Storage, THEN THE Quick_Links SHALL render an empty panel with no errors.

---

### Requirement 8: Quick Links — Add and Delete Links

**User Story:** As a user, I want to add and remove quick-link shortcuts, so that I can customise the panel to reflect my current favourite sites.

#### Acceptance Criteria

1. THE Quick_Links SHALL provide input fields for a link label and a link URL, and a submit control for adding a new Link.
2. WHEN the user submits a non-empty label and a non-empty URL, THE Quick_Links SHALL append the new Link to the panel and persist the updated Link collection to Local_Storage.
3. IF the user submits with an empty label or an empty URL, THEN THE Quick_Links SHALL reject the submission and retain focus on the first empty field.
4. THE Quick_Links SHALL provide a delete control for each Link.
5. WHEN the user activates the delete control for a Link, THE Quick_Links SHALL remove the Link from the panel and persist the updated Link collection to Local_Storage.

---

### Requirement 9: Data Persistence Architecture

**User Story:** As a user, I want all my data stored locally in my browser, so that my information remains private and the app works without an internet connection after the initial load.

#### Acceptance Criteria

1. THE Dashboard SHALL store all Task data in Local_Storage under the key `todo_life_tasks`.
2. THE Dashboard SHALL store all Link data in Local_Storage under the key `todo_life_links`.
3. THE Dashboard SHALL store all data as JSON-serialised strings.
4. FOR ALL valid in-memory data collections, serialising to JSON and then deserialising SHALL produce a collection equivalent to the original (round-trip property).
5. IF Local_Storage is unavailable or throws an error during a write operation, THEN THE Dashboard SHALL display a non-blocking warning message to the user without losing the in-memory data.

---

### Requirement 10: Browser Compatibility and Deployment

**User Story:** As a user, I want the Dashboard to work in any modern browser without installation, so that I can use it on any device with a browser.

#### Acceptance Criteria

1. THE Dashboard SHALL render and function correctly in the current stable release of Chrome, Firefox, Edge, and Safari.
2. THE Dashboard SHALL operate as a standalone web page loaded from the local file system (via `file://` protocol) without requiring a web server.
3. THE Dashboard SHALL use only one CSS file located at `css/style.css` and one JavaScript file located at `js/app.js`.
4. THE Dashboard SHALL load and become interactive within 2 seconds on a standard desktop machine with no network dependency after the initial file load.

---

### Requirement 11: Visual Design and Usability

**User Story:** As a user, I want a clean, readable, and visually organised interface, so that I can use the Dashboard comfortably for extended periods without cognitive strain.

#### Acceptance Criteria

1. THE Dashboard SHALL apply a clear visual hierarchy that distinguishes the Greeting_Widget, Focus_Timer, Todo_List, and Quick_Links as separate sections.
2. THE Dashboard SHALL use a readable font size of at least 14px for body text and task labels.
3. THE Dashboard SHALL provide sufficient colour contrast between text and background colours to meet WCAG 2.1 AA contrast ratio requirements (minimum 4.5:1 for normal text).
4. THE Dashboard SHALL be responsive and remain usable at viewport widths between 320px and 1920px.
5. WHILE the Focus_Timer is counting down, THE Dashboard SHALL display the remaining time in a prominent, easily readable format.
