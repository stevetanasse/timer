# Timer App – Claude Code Tickets

Nov 5, 2026 · @Steve

## How to use this document

Give Claude Code one ticket per session, in order from T1 to T10. Each ticket says what it depends on, what is in and out of scope, its acceptance criteria, and a Verify step that proves it is done.

1. Create an empty folder such as `C:\src\timer` and start Claude Code in it.
2. Hand it T1. T1 saves the Functional specification and Project conventions sections below into the repo as `docs/SPEC.md` and `CLAUDE.md`, so every later session reads them automatically.
3. For each ticket, paste the whole ticket and ask Claude Code to plan first, then implement, then run the Verify step and report the results.
4. Review the changes, then commit before starting the next ticket. One ticket = one commit (or one pull request once T10 is in place).
5. T7 to T10 touch Azure. Claude Code writes the scripts and explains each command; you run them yourself, so you see every resource being created and nothing is billed without you knowing.

If a ticket and `docs/SPEC.md` ever disagree, the spec wins. Tell Claude Code to stop and ask rather than guess.

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

&#91;embedded content: timer states · 4 states, 5 transitions\]

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

## Project conventions (CLAUDE.md)

T1 saves this text as `CLAUDE.md` at the repo root. Claude Code reads that file at the start of every session, so these rules apply to every ticket.

```markdown
# Timer

A countdown timer web app. It is a learning project for Docker and Azure Container Apps.
The full behavior is in docs/SPEC.md, which is the source of truth. If a request conflicts with it, stop and ask.

## Stack
- Vite + TypeScript (strict mode), no UI framework, plain CSS
- Vitest for unit tests
- Multi-stage Dockerfile; nginx-unprivileged serves the built site on port 8080

## Commands
- npm run dev        local dev server
- npm test           unit tests (Vitest, run once)
- npm run typecheck  tsc --noEmit
- npm run build      production build into dist/

## Code layout
- src/time.ts    parse and format HH:MM:SS (pure functions)
- src/timer.ts   timer state machine (pure, clock injected)
- src/ui.ts      DOM wiring only
- src/main.ts    entry point
- src/styles.css styles and the flash animation
Logic modules never touch the DOM. UI code holds no timing logic.

## Rules
- The host is Windows 11. Scripts are PowerShell (.ps1).
- Files used inside the container must have LF line endings; .gitattributes enforces this.
- Do not add runtime dependencies without asking.
- Never run commands that create, change or delete Azure resources.
  Write them into scripts under infra/, explain each command, and let the user run them.
- Never commit secrets, keys or connection strings.
- Finish every ticket with: tests pass, typecheck passes, build succeeds,
  then a short summary of what changed and how to verify it.
```

## Tickets T1 to T5: the app

These five tickets produce a finished timer that runs with `npm run dev`. Nothing in them touches Docker or Azure.

### T1. Project scaffold

**Goal:** a working, empty Vite + TypeScript project in a Git repository. **Depends on:** nothing.

**Scope**

- `git init`, then scaffold in place with the Vite `vanilla-ts` template. Remove the demo counter code and sample assets.
- Add Vitest. Add npm scripts `dev`, `test` (run once, not watch), `typecheck` and `build`.
- Add `.gitignore` (`node_modules`, `dist`) and `.gitattributes` with `* text=auto eol=lf` and `*.ps1 text eol=crlf`.
- Save `docs/SPEC.md` from this document's Decisions and Functional specification sections, and `CLAUDE.md` from the Project conventions section.
- Add a short `README.md`: what the app is and how to run it locally.
- The page shows only the heading "Timer" for now.

**Out of scope:** timer logic, styling, Docker.

**Acceptance criteria**

- [ ] `npm run dev` serves a page titled "Timer"
- [ ] `npm test` runs and one placeholder test passes
- [ ] `npm run typecheck` and `npm run build` succeed
- [ ] No Vite demo code remains

**Your step after T1:** create an empty repository named `timer` on github.com (no README or license), then ask Claude Code for the commands to add it as `origin` and push.

### T2. Time parsing and formatting

**Goal:** pure functions that turn text into seconds and seconds into text, fully tested. **Depends on:** T1.

**Scope:** `src/time.ts` exports:

- `parseDuration(text)` returning either `{ ok: true, seconds }` or `{ ok: false, error }`
- `formatDuration(seconds)` returning zero-padded `HH:MM:SS`
- `MAX_SECONDS = 43200` and the four error messages from the spec as named constants

**Acceptance criteria**

- [ ] Every row of the spec's Input validation table is a test case
- [ ] Formatting tests for 0, 59, 60, 3599, 3600 and 43200 seconds
- [ ] Round trip: formatting a parsed valid input gives back the same text
- [ ] `"  00:03:00 "` (with spaces) is valid
- [ ] The module has no DOM access

**Verify:** `npm test`.

### T3. Timer state machine

**Goal:** a DOM-free timer implementing every state and transition in the spec, tested with a fake clock. **Depends on:** T2.

**Scope:** `src/timer.ts`:

- States `idle`, `running`, `paused`, `alarming`; actions `play(text)`, `pause()`, `stop()`, `reset()`, `tick()`.
- The clock is injected (`now: () => number`) so tests control time. The machine does not create timers itself; the UI calls `tick()`.
- Running stores the end timestamp. Paused stores the exact remaining milliseconds and the text shown at the moment of pausing. The machine also holds the last entered time: set whenever play starts from typed text, cleared by reset.
- `getSnapshot()` returns: state, text to display, error or null, primary action (`play`, `pause` or `stop`), whether the input is editable, and whether the page is flashing. An `onChange(listener)` subscription fires on every change.

**Acceptance criteria**

- [ ] One test per transition in the spec's Transitions list
- [ ] Invalid `play` leaves the state unchanged and sets the error
- [ ] Pausing at 2,750 ms remaining and resuming with unchanged text resumes at 2,750 ms
- [ ] Editing the text while paused and pressing play starts from the new value
- [ ] After `stop()`, `play('00:00:00')` starts the last entered time; after `reset()`, the same call shows the "greater than 00:00:00" error
- [ ] 2,001 ms remaining displays `00:00:03`; reaching 0 displays `00:00:00` and enters `alarming`
- [ ] A clock jump far past the end time (laptop sleep) goes straight to `alarming`
- [ ] `reset()` from every state gives `idle`, `00:00:00`, no error

**Verify:** `npm test`.

### T4. Page layout and controls

**Goal:** the working page, everything except the flash effect. **Depends on:** T3.

**Scope**

- Markup in `index.html`: heading, input, error area, primary button and Reset button with inline SVG icons for play, pause, stop and reset.
- `src/ui.ts`: clicks call the machine; a 250 ms interval calls `tick()` while running; `visibilitychange` triggers an immediate tick; the page renders from the snapshot.
- Do not overwrite the input while the user is typing in Idle or Paused. Typing clears the error.
- Tab title as described in the spec.
- Layout: centered, large digits, works at phone width, visible keyboard focus.
- In Alarming, only the button and title change for now; the overlay is T5.

**Acceptance criteria**

- [ ] `00:00:05` then Play counts down to zero; the button shows Pause; the input is read-only
- [ ] Pause then Play resumes; editing while paused then Play starts the new value
- [ ] Each invalid input from the spec shows its message below the input; the message clears on edit
- [ ] At zero the button shows Stop; Stop returns to `00:00:00` and the Play icon; Play then restarts the last entered time
- [ ] Reset works in all four states
- [ ] The tab title updates as specified
- [ ] No errors in the browser console

**Verify:** walk through the criteria in `npm run dev`; `npm test`, `npm run typecheck` and `npm run build` pass.

### T5. Flashing overlay

**Goal:** the full-page flash on expiry, exactly as the spec describes. **Depends on:** T4.

**Scope**

- Add the overlay element when entering Alarming and remove it when leaving.
- CSS along these lines:

```css
@keyframes flash { 0%, 100% { opacity: 0.5; } 50% { opacity: 0; } }
.flash-overlay {
  position: fixed; inset: 0; z-index: 1000;
  background: rgb(128 128 128);
  pointer-events: none;
  animation: flash 2s ease-in-out infinite;
}
```

- Write `docs/QA.md`: a manual test checklist combining the criteria from T4 and T5. You will reuse it after each deployment.

**Acceptance criteria**

- [ ] At zero the page is immediately 50% gray, fades smoothly to normal and back every 2 seconds
- [ ] Stop and Reset are clickable while flashing, and both end it with no leftover tint
- [ ] Flashing is still running when you switch back from another tab
- [ ] No sound of any kind

**Verify:** run a 3-second timer, watch two cycles, click Stop. Repeat and click Reset instead.

## Ticket T6: the container

T6 packages the finished app into a small, non-root nginx image that you run on your laptop exactly as Azure will run it.

### T6. Dockerfile, nginx and local container run

**Goal:** a multi-stage image that serves the built site on port 8080 and passes the QA checklist locally. **Depends on:** T5.

**Scope**

- `Dockerfile`, commented line by line so you can learn from it:
  - Build stage `FROM node:<current LTS>-alpine AS build`: copy `package.json` and `package-lock.json` first, `npm ci`, then copy the source, run `npm test`, then `npm run build`.
  - Runtime stage `FROM nginxinc/nginx-unprivileged:<stable>-alpine`: copy `nginx.conf` and the `dist/` folder from the build stage, `EXPOSE 8080`.
  - Pin both base images to a specific version tag, never `latest`.
- `nginx.conf`: listen on 8080; serve `dist/`; `GET /healthz` returns 200 with body `ok` and no access log; long-lived caching for hashed files in `/assets/`; `no-cache` for `index.html`; gzip on.
- `.dockerignore`: `node_modules`, `dist`, `.git`, `infra`, `docs`, `*.md`.
- `scripts/docker-local.ps1`: builds with `--platform linux/amd64 -t timer:local`, then runs with `--rm -p 8080:8080`.
- A "Run in Docker" section in `README.md`.

**Out of scope:** pushing images anywhere.

**Acceptance criteria**

- [ ] The build succeeds, and the app at `http://localhost:8080` passes every item in `docs/QA.md`
- [ ] `curl.exe -i http://localhost:8080/healthz` returns `200` and `ok`
- [ ] `docker exec <container> id` shows a non-root user
- [ ] Making one unit test fail makes `docker build` fail (then revert it)
- [ ] Rebuilding after a CSS-only change reuses the cached `npm ci` layer
- [ ] The final image size is reported, and no Node.js or `node_modules` is inside it

**Verify:** run `scripts/docker-local.ps1` and walk through `docs/QA.md`.

**What you learn here:** layer caching (why the package files are copied before the source), multi-stage builds (the final image holds only nginx and static files), and running as a non-root user.

## Tickets T7 and T8: deploy with the Azure CLI

These two tickets put the app on a public HTTPS URL using Azure CLI scripts that you run by hand, one command group at a time.

| Resource | Example name | What it is for |
| --- | --- | --- |
| Resource group | `rg-timer` | Holds everything; deleting it removes the whole project |
| Container registry | `acratimer` + a unique suffix | Stores your images; the name must be globally unique, lowercase letters and digits |
| Log Analytics workspace | `log-timer` | Receives the app's console and system logs |
| Container Apps environment | `cae-timer` | The shared boundary and network that apps run in |
| Container app | `ca-timer` | Runs your image; each deployment creates a new revision |

### T7. Provision the Azure resources

**Goal:** a resource group, registry, log workspace and Container Apps environment, created by scripts you can rerun safely. **Depends on:** T6.

**Scope**

- `infra/cli/config.ps1`: every name above, the subscription ID and a region near you (for example `westus2`) in one place. Other scripts dot-source it.
- `infra/cli/00-prereqs.ps1`: `az login`, `az account set`, add or upgrade the `containerapp` CLI extension, and register the `Microsoft.App`, `Microsoft.OperationalInsights` and `Microsoft.ContainerRegistry` providers, waiting until each is registered.
- `infra/cli/01-provision.ps1`: create the resource group, the registry (Basic tier, admin user disabled), the Log Analytics workspace and the environment linked to it.
- `infra/README.md`: what each resource is and what each command does.

**Acceptance criteria**

- [ ] Every script sets `$ErrorActionPreference = 'Stop'` and checks `$LASTEXITCODE` after each `az` call, because `az` failures do not stop PowerShell by themselves
- [ ] Running `01-provision.ps1` a second time succeeds and changes nothing
- [ ] `az resource list -g rg-timer -o table` shows the registry, workspace and environment
- [ ] No keys or secrets are printed or committed

**Your step:** run `00-prereqs.ps1`, then `01-provision.ps1`, and look at the new resources in the Azure portal.

### T8. Push the image and deploy the app

**Goal:** the timer live on its `azurecontainerapps.io` URL, and a repeatable way to ship a new version. **Depends on:** T7.

**Scope**

- `infra/cli/02-push-image.ps1`: `az acr login`, build with `--platform linux/amd64`, tag as `<registry>.azurecr.io/timer:<git short SHA>`, push. Never rely on the `latest` tag.
- `infra/cli/03-deploy-app.ps1`:
  - First run creates the app with a system-assigned managed identity that pulls from the registry (`--registry-identity system`), so no registry password exists anywhere.
  - External ingress to target port 8080, 0.25 CPU and 0.5 GiB memory, minimum 0 and maximum 1 replica.
  - Later runs update the image to the new tag, which creates a new revision.
  - Prints the app URL at the end.
- `infra/cli/logs.ps1`: streams the app's console logs.
- `infra/cli/99-teardown.ps1`: deletes the resource group after you type its name to confirm.

**Acceptance criteria**

- [ ] The HTTPS URL loads the timer and passes every item in `docs/QA.md`
- [ ] `https://<app URL>/healthz` returns `ok`
- [ ] Change the page heading, rerun `02` and `03`: the change is live and `az containerapp revision list` shows two revisions
- [ ] After some idle minutes the app scales to zero replicas, and the next request wakes it after a short delay
- [ ] The registry admin user is still disabled

**Your step:** run `02` and `03`, open the URL, then run through `docs/QA.md` on your phone as well.

**What you learn here:** registries and image tags, managed identity instead of passwords, revisions, ingress, and scale to zero.

## Tickets T9 and T10: infrastructure as code and CI/CD

These two tickets replace the hand-run scripts with a Bicep template and a GitHub Actions pipeline, so a merge to `main` deploys automatically.

### T9. Bicep template

**Goal:** the same infrastructure described as code, proven by deploying it into a fresh resource group. **Depends on:** T8.

**Scope**

- `infra/bicep/main.bicep` (resource group scope) and `main.bicepparam`: Log Analytics workspace, registry (Basic, admin disabled), Container Apps environment, and the container app with ingress on 8080 and 0 to 1 replicas.
- Use a user-assigned managed identity that is granted `AcrPull` before the app is created. This avoids the race where a new app tries to pull before its own identity has permission.
- Add HTTP liveness and readiness probes on `/healthz`, port 8080. The CLI deployment relied on the platform defaults.
- A `deployApp` parameter: the first run creates only the infrastructure so you can push an image, the second run creates the app.
- `infra/bicep/deploy.ps1`: runs `az deployment group what-if` and shows the result before `az deployment group create`. Target resource group `rg-timer-bicep`.

**Acceptance criteria**

- [ ] `az bicep build` reports no warnings, or each remaining warning is explained
- [ ] The app in `rg-timer-bicep` passes `docs/QA.md` and `/healthz` returns `ok`
- [ ] The probes are visible on the app in the Azure portal
- [ ] Running `deploy.ps1` again with no changes shows no resource changes in what-if
- [ ] `infra/cli/config.ps1` and the README now point at the Bicep resource group

**Your step:** run `deploy.ps1` twice as described, test the app, then run `99-teardown.ps1` on the original `rg-timer` so only one copy remains.

### T10. GitHub Actions CI/CD

**Goal:** checks on every pull request, and every push to `main` builds, pushes and deploys a new revision. **Depends on:** T9.

**Scope**

- `.github/workflows/ci.yml` on pull requests and pushes: `npm ci`, typecheck, test, build, then `docker build` without pushing.
- `.github/workflows/deploy.yml` on push to `main` and manual dispatch:
  - Sign in with `azure/login` using OpenID Connect (a federated credential), so no client secret exists anywhere.
  - Build and push the image tagged with the commit SHA; update the container app to that tag.
  - Smoke test: request `/healthz` until it returns 200, or fail the run after a time limit.
  - Write the image tag and app URL to the run summary.
- `infra/github/setup-oidc.ps1`: creates a Microsoft Entra app registration and service principal, a federated credential for `repo:<your user>/timer:ref:refs/heads/main`, and role assignments scoped to the Bicep resource group only (`AcrPush` on the registry, permission to update the app). It prints the values to add as repository variables: `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`.

**Acceptance criteria**

- [ ] A pull request with a failing test shows a red check and cannot be merged
- [ ] Merging a heading change deploys automatically, and the new revision serves it
- [ ] No passwords or client secrets are stored in GitHub
- [ ] No role assignment is wider than the resource group

**Your step:** run `setup-oidc.ps1`, add the three repository variables, and turn on branch protection for `main` requiring the CI check.

**What you learn here:** what-if previews, identity ordering in templates, passwordless sign-in from GitHub to Azure, and least-privilege role scoping.

## Cost and cleanup

The registry is the only resource that bills while nothing is happening; deleting the resource group stops all charges.

- **Container registry (Basic):** billed per day whether or not you use it. This is the main standing cost of the project.
- **Container app:** with minimum replicas set to 0 it bills only while handling requests, which for a personal timer is close to nothing.
- **Log Analytics:** billed by data ingested; a small static site sends very little.
- **Budget alert:** set a low monthly budget with an email alert on the subscription in the Azure portal (Cost Management, Budgets) before T7.
- **Cleanup:** when you finish, run `99-teardown.ps1` against whichever resource group is current. Also delete the Entra app registration that T10 created.

Check current prices on the Azure pricing pages for Container Registry and Container Apps; this document does not quote figures.
