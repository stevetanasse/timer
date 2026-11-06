# Timer – Specification

Source of truth for the app. Copied from the "Decisions and environment" and "Functional specification" sections of docs/TICKETS.md.

## Decisions and environment

The app is a static Vite + TypeScript site, built in a multi-stage Docker image, served by nginx, and deployed to Azure Container Apps from Azure Container Registry.

| Area | Decision |
| --- | --- |
| UI | Vite, vanilla TypeScript, plain CSS, no UI framework |
| Tests | Vitest unit tests for parsing, formatting and the timer state machine; no browser end-to-end tests |
| Container | Multi-stage: Node (current LTS) builds `dist/`; `nginxinc/nginx-unprivileged` (Alpine) serves it on port 8080 as a non-root user |
| Registry | Azure Container Registry, Basic tier |
| Hosting | Azure Container Apps, Consumption plan, external ingress, default `*.azurecontainerapps.io` URL, no login, min replicas 0, max replicas 1 |
| Deployment path | Azure CLI scripts first (T7, T8), then Bicep (T9), then GitHub Actions (T10) |
| Source control | A new GitHub repository on your existing account |
| Laptop | Windows 11 Pro, Intel Core i9-13950HX (x64), Docker 29.9.2, Azure CLI 2.80.0 |
| Shell | PowerShell for all scripts |

Your laptop builds `linux/amd64` images natively, which is what Container Apps runs, so no cross-platform build flags are needed. The tickets still pass `--platform linux/amd64` explicitly so the scripts also work on other machines.

## Functional specification

The timer has four states, Idle, Running, Paused and Alarming, and every click moves it between them as described here.

| From | Event | To |
| --- | --- | --- |
| Idle | Play clicked, valid time | Running |
| Running | Pause clicked | Paused |
| Paused | Play clicked, valid time | Running |
| Running | Remaining time reaches zero | Alarming |
| Alarming | Stop clicked | Idle |
| Any state | Reset clicked | Idle |

Only Running counts down; Paused holds the exact remaining time until Play is clicked again.

### Screen

- One text input, initial value `00:00:00`, large monospace text.
- A primary button whose icon and label change with the state (Play, Pause or Stop), and a Reset button.
- An error message area directly below the input, announced to screen readers (`role="alert"`).
- Icons are inline SVG, not emoji characters, because emoji such as the pause symbol render inconsistently on Windows. Each button has an `aria-label` and `title`.

### What each state shows

| State | Textbox | Primary button | Page |
| --- | --- | --- | --- |
| Idle | Editable | Play | Normal |
| Running | Read-only, shows remaining time | Pause | Normal |
| Paused | Editable, shows remaining time | Play | Normal |
| Alarming | Read-only, shows `00:00:00` | Stop | Flashing |

Reset is enabled in every state.

### Transitions

- **Idle, Play clicked:** validate the text. Invalid shows the error and stays Idle. Valid starts the countdown (Running).
- **Running, Pause clicked:** freeze the countdown; the textbox keeps the remaining time (Paused).
- **Paused, Play clicked:** if the text is unchanged since pausing, resume from the exact remaining time. If the user edited it, validate and start from the new value; invalid shows the error and stays Paused.
- **Running reaches zero:** start flashing (Alarming).
- **Alarming, Stop clicked:** stop flashing, keep the last entered time, set the textbox to `00:00:00` (Idle).
- **Any state, Reset clicked:** stop the countdown and any flashing, clear the error, set the textbox to `00:00:00` (Idle).

**Reusing the last time.** The timer remembers the last time the user entered: the last valid typed value that started a countdown. Resuming from Pause does not change it. After Stop the textbox reads `00:00:00`, and clicking Play without typing starts the remembered time again; typing a different time first starts the new one. Reset clears the remembered time, so after Reset (or on first page load) Play on `00:00:00` shows the "greater than 00:00:00" error.

### Input validation

Validation runs when Play is clicked. Leading and trailing spaces are trimmed, then the text must be exactly `HH:MM:SS` with two digits in each part. The error clears as soon as the user edits the text.

| Input | Result | Error message shown |
| --- | --- | --- |
| `00:03:00` | Valid, 180 seconds | none |
| `12:00:00` | Valid, 43,200 seconds (the maximum) | none |
| `3:00`, `00:3:00`, `180`, `abc`, empty | Invalid | Enter the time as HH:MM:SS, for example 00:03:00. |
| `00:60:00`, `00:00:75` | Invalid | Minutes and seconds must be between 00 and 59. |
| `12:00:01`, `13:00:00`, `99:59:59` | Invalid | The maximum time is 12:00:00. |
| `00:00:00` | Invalid | Enter a time greater than 00:00:00. |

### Countdown accuracy

- Remaining time is computed from an end timestamp (`Date.now()`), never by subtracting one per tick. This keeps it accurate when the browser throttles background tabs or the laptop sleeps.
- The display rounds remaining time up to the whole second, so it shows `00:00:01` during the last second and `00:00:00` exactly at expiry.
- The display refreshes about 4 times a second, and immediately when the tab becomes visible again.
- The browser tab title shows the remaining time while Running or Paused (`00:02:41 - Timer`), `Time's up - Timer` while Alarming, and `Timer` otherwise.

### Flashing (Alarming state)

- A full-viewport gray overlay (`position: fixed; inset: 0`) covers the whole page, including the controls, with `pointer-events: none` so Stop and Reset remain clickable.
- A CSS animation moves the overlay's opacity smoothly from 0.5 to 0 and back to 0.5, one full cycle every 2 seconds, `ease-in-out`, repeating indefinitely. It starts at 0.5 the moment the timer reaches zero.
- Flashing continues until Stop or Reset. There is no sound.
- In every other state the overlay is removed from the page, not just made transparent.

