# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

"The Naminator" — an AI-powered name combination generator. Users sign in with Google OAuth, enter two names, and Claude generates creative combined names with goodness scores (0-5). Results are persisted per-user in PostgreSQL.

## Commands

- `npm run dev` — start Next.js dev server (http://localhost:3000)
- `npm run build` — build for production (runs `prisma generate` first)
- `npm run lint` — run ESLint
- `npm run db:push` — push Prisma schema to database (no migrations)
- `npm run db:migrate` — create and apply a Prisma migration
- `npm run db:studio` — open Prisma Studio (visual DB browser)

After changing `prisma/schema.prisma`, run `npm run db:push` (dev) or `npm run db:migrate` (production) then `npx prisma generate` to update the client.

### Testing

- `npm test` — vitest watch mode
- `npm run test:run` — single pass (use for CI or verifying changes)
- `npm run test:coverage` — run with coverage report

Test files live in `src/**/__tests__/` directories. To run a single test file:

```sh
npx vitest run src/lib/__tests__/anthropic.test.ts
```

Test files use `// @vitest-environment node` at the top when they test server-side code (API routes, lib functions); component tests default to jsdom.

`src/test/setup.ts` runs before every test: it imports `@testing-library/jest-dom` matchers and globally suppresses `console.error` (so test output is clean by default).

## Architecture

Next.js 16 App Router with TypeScript, Tailwind CSS 4, Prisma 6, Auth.js v5, and the Anthropic SDK.

**Authentication flow:** Auth.js v5 with Google OAuth, configured in `src/auth.ts` using the Prisma adapter. The middleware (`src/middleware.ts`) protects all routes — unauthenticated users are redirected to `/login`, and logged-in users visiting `/login` are redirected to `/`. The session callback attaches `user.id` to the session object. `src/types/next-auth.d.ts` extends the Auth.js `Session` type to include `user.id`.

**API routes:** Single resource at `src/app/api/name-combinations/route.ts` with POST (generate new combinations via Claude and save) and GET (fetch user's history). Both endpoints require authentication via `auth()`.

**AI integration:** `src/lib/anthropic.ts` exports `generateNameCombinations()` which prompts Claude (claude-sonnet-4-20250514) to return a JSON array of `{name, goodness}` objects. The response is parsed and validated server-side. Goodness scores are clamped to [0, 5] and rounded to one decimal.

**Data model:** `NameCombinationSet` (two input names, userId) has many `GeneratedName` (name, goodness score). Auth.js models (User, Account, Session, VerificationToken) are also in the Prisma schema.

**Page structure:** The main page (`src/app/page.tsx`) is a server component that fetches history via Prisma and passes serialized data to `NameForm` (client component). `NameForm` handles form submission and renders `ResultCard` components.

## Testing Architecture

Tests mock all external dependencies at the module boundary:

- **`@anthropic-ai/sdk`** — mocked with `vi.hoisted()` + a regular function (not arrow) since `src/lib/anthropic.ts` calls `new Anthropic()` at module load time and arrow functions can't be constructors
- **`@/auth`** — mocked to return a fake session or `null`
- **`@/lib/prisma`** — mocked with fake `nameCombinationSet.create` / `findMany` implementations
- **`@/lib/anthropic`** — mocked in route tests so they test HTTP behavior independently of AI logic
- **`global.fetch`** — mocked with `vi.stubGlobal` in `NameForm` and `ResultCard` component tests

## Environment Variables

Required in `.env`: `DATABASE_URL` (Neon PostgreSQL), `AUTH_SECRET`, `AUTH_GOOGLE_ID`, `AUTH_GOOGLE_SECRET`, `ANTHROPIC_API_KEY`.

Note: Neon `DATABASE_URL` values typically require `?sslmode=require` appended to the connection string.

## Known Quirks

- **`NEXT_REDIRECT` errors in tests**: Auth.js uses Next.js redirects internally; these are expected and not bugs.
- **Prisma client not found after install**: Run `npx prisma generate` — the `postinstall` script handles this automatically on `npm install`.
