# Working agreement for AI agents

Every AI agent working on this repo reads this file first and follows it. It fixes the architecture, the patterns and the rules so that every feature looks like it was written by the same person.

## Product

A Twitter/X clone: custom auth, profiles, tweets (max 280 chars), timeline of followed users with pagination, follow/unfollow, like/unlike with counters, followers/following lists, user search, notifications (bonus), mobile-first responsive UI.

## Stack (do not change without updating the README)

| Layer | Choice |
|---|---|
| Runtime | Node.js 22 LTS (`.nvmrc`), npm workspaces |
| Language | TypeScript everywhere, `strict: true` |
| API | Fastify 5 + `@fastify/cookie` + `@fastify/cors` + zod for input validation |
| DB | SQLite through Prisma ORM (`apps/api/prisma/schema.prisma`) |
| Auth | Custom: `bcryptjs` password hashes, random opaque session tokens stored hashed (SHA-256) in a `Session` table, `HttpOnly` + `SameSite=Lax` cookie `sid`, 30-day expiry. No JWT, no third-party auth. |
| Web | React 18 + Vite + React Router + TanStack Query + Tailwind CSS (mobile-first) |
| Tests | Vitest (+ `@vitest/coverage-v8`) for API unit/integration; Vitest + Testing Library + MSW for web; Playwright for e2e |

## Repo layout

```text
apps/api/      Fastify API
  src/app.ts             buildApp(): registers plugins and routes, used by server.ts and tests
  src/server.ts          starts the server
  src/config.ts          reads env vars (zod-validated)
  src/db.ts              Prisma client singleton
  src/plugins/auth.ts    session lookup → request.user; requireAuth preHandler
  src/modules/<feature>/ <feature>.routes.ts · <feature>.service.ts · <feature>.schemas.ts
  src/lib/               small shared helpers (errors, pagination, password, tokens)
  prisma/schema.prisma · prisma/seed.ts
  test/unit/             pure logic and validation
  test/integration/      HTTP tests through app.inject() against a fresh test DB
apps/web/      React SPA
  src/api/               typed fetch client, one file per feature
  src/features/<feature>/ components + hooks for that feature
  src/components/        shared UI (Button, Avatar, Layout, TweetCard…)
  src/pages/             route-level pages
  src/test/              MSW handlers + test utils
e2e/           Playwright tests (run against the real API + web)
```

## API conventions

- All routes under `/api`. JSON in, JSON out. Errors are `{ "error": { "code": "SNAKE_CASE", "message": "Human text" } }` with the right status (400 validation, 401 not logged in, 403 not yours, 404 missing, 409 conflict).
- Validate every body/query/param with zod in `<feature>.schemas.ts`; never trust the client.
- Business logic lives in `<feature>.service.ts` (no Fastify types there); routes only parse, call the service and shape the response.
- Pagination is cursor-based: `?cursor=<id>&limit=<n>` (default 20, max 50) → `{ items, nextCursor }`. Order by `createdAt DESC, id DESC`.
- Usernames: 3–15 chars, `[a-zA-Z0-9_]`, unique case-insensitively (store lowercase in `usernameLower`). Emails stored lowercase.
- Tweets: trimmed text, 1–280 characters (count with `[...text].length` so emojis count as one).
- Never return `passwordHash` or session tokens. Users are serialized through one `toPublicUser()` helper.

## Web conventions

- Mobile-first Tailwind: base styles for < 640px, then `sm:` (≥ 640) and `lg:` (≥ 1024). Bottom nav on mobile, left sidebar on desktop.
- Server state only through TanStack Query hooks in `src/features/*/hooks.ts`; no fetching inside components.
- Every interactive element has an accessible name; tests locate by role/label/text, not CSS.

## Naming and style

- camelCase for variables/functions, PascalCase for components/types, kebab-case for file names except React components (`TweetCard.tsx`).
- No commented-out code. Comments only where the *why* is not obvious.
- Prettier defaults (2 spaces, double quotes, trailing commas). ESLint must pass.

## Tests (the rubric weighs testing 25%)

- API coverage ≥ 85% lines/branches on `apps/api/src` (enforced by Vitest thresholds).
- Every endpoint gets integration tests for: happy path, validation error, auth required, and ownership/permission where it applies.
- Assertions check real outcomes (status, body fields, DB state), never just "it returned something".
- Web: integration tests with Testing Library + MSW for login, create tweet and follow at minimum.
- E2E (Playwright): at least the full auth flow (register → logout → login → protected page).
- Tests are written in the same commit as the feature, never "all tests at the end".

## Commits (the history is evaluated)

- Small, coherent commits that tell the story; Conventional Commits: `feat(api): …`, `feat(web): …`, `test(api): …`, `chore: …`, `docs: …`, `fix: …`.
- Run the relevant tests before each commit; never commit red tests or a broken build.
- Never squash, never rewrite pushed history, never commit `.env` or databases.
- Git identity is already configured in the repo; do not change it.

## Definition of done for a feature

1. API endpoint(s) + zod schemas + service + integration tests (+ unit tests for pure logic).
2. Web UI wired through a query hook, responsive at 375px, 768px and 1280px.
3. Web integration test for the main flow of the feature.
4. `npm test` green and coverage thresholds met.
