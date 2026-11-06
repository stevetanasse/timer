# ACA Timer

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
