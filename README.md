# Hotel Booking

A single-property hotel booking app. See [`GLOSSARY.md`](./GLOSSARY.md) for the domain terms and
[`docs/adr/0001-v1-stack-and-scope.md`](./docs/adr/0001-v1-stack-and-scope.md) for the v1 design.

## Stack

Next.js 16 (App Router, Turbopack) · React 19 · TypeScript · Tailwind CSS · shadcn/ui · Vitest.

## Getting started

Package manager: **pnpm**. To install and run:

```bash
pnpm install
pnpm dev
```

Open [http://localhost:3000](http://localhost:3000).

## Scripts

| Script | What it does |
|--------|--------------|
| `pnpm dev` | Start the dev server (Turbopack) |
| `pnpm build` | Production build |
| `pnpm start` | Run the production build |
| `pnpm lint` | ESLint (Flat Config) over the repo |
| `pnpm typecheck` | Generate Next route types and run `tsc --noEmit` |
| `pnpm test` | Run Vitest once |
| `pnpm test:watch` | Run Vitest in watch mode |

## Project layout

```
src/
  app/                  Next.js App Router
  components/ui/        shadcn/ui components
  lib/                  app-level helpers
docs/
  adr/                  Architecture Decision Records
  agents/               Agent-skills configuration (AGENTS.md pointers)
AGENTS.md               Agent instructions (read first)
GLOSSARY.md             Domain vocabulary
```

## Issues and tickets

This project uses [GitHub Issues](https://github.com/TheZenMind07/hotel-booking/issues) with the
default five-role triage vocabulary (`needs-triage`, `needs-info`, `ready-for-agent`,
`ready-for-human`, `wontfix`). See `docs/agents/issue-tracker.md` for how to create and read
tickets.
