# Project Brief: Event Timestamp Recorder PWA

## Overview

A single-file Progressive Web App (PWA) for Android that records timestamps of button presses representing observed events.
The user configures up to 8 buttons, each representing a named event.
Button presses are logged with timestamps and the log can be exported as a CSV file.

## Deliverables

A single self-contained `index.html` file with all CSS and JavaScript inline.
No build tools, no frameworks, no external dependencies.
Optionally a `manifest.json` for PWA installability.

## Core Features

### Main Screen

- Display between 1 and 8 event buttons (only configured/active buttons are shown)
- Each button displays its user-configured label
- Pressing a button records an entry: `{ timestamp, eventName }`
- Timestamp should be recorded in ISO 8601 format (e.g. `2026-05-23T14:32:01.452Z`)

### Logs Screen

- An Export CSV button downloads the log as a `.csv` file with two columns: timestamp and event. The filename should include the current date, e.g. `events-2026-05-23.csv`
- A Clear button clears the in-memory log (with a confirmation prompt)
- A scrollable log of recorded events is displayed on screen, newest entries at the top

### Settings Screen

- Accessible via a Settings button or gear icon on the main screen
- A list of 8 button slots, each with:
    - A toggle to enable/disable the button
    - A text field for the button label (shown on the button)
    - A text field for the event name (recorded in the log)
    - A selector to choose the button color
- Settings are persisted to `localStorage` so they survive page reloads
- A Back or Done button returns to the main screen

### UX Requirements

- Designed for mobile (Android phone). Touch-friendly: buttons should be large enough to tap comfortably
- Responsive layout that works in portrait orientation
- No external network requests — fully self-contained and works offline
- Clean, functional UI — no need to be fancy, but should be readable and uncluttered

### PWA (Optional but Recommended)

- Add a `manifest.json` referencing the HTML file so the app can be added to the Android home screen via Chrome's "Add to Home Screen"
- Add a minimal service worker for offline support

### Out of Scope

- User accounts, sync, or cloud storage
- Multiple sessions or named sessions
- Any backend