# Timer

A countdown timer web app: enter a time as `HH:MM:SS`, press Play, and the page flashes when it reaches zero.
It is a learning project for Docker and Azure Container Apps.

- Behavior: [docs/SPEC.md](docs/SPEC.md)
- Work plan: [docs/TICKETS.md](docs/TICKETS.md)

## Run locally

Requires Node.js (current LTS).

```powershell
npm install
npm run dev        # dev server, open the URL it prints
```

Other commands:

```powershell
npm test           # unit tests (Vitest, run once)
npm run typecheck  # tsc --noEmit
npm run build      # production build into dist/
```
