# [YOUR_PROJECT_NAME] — Claude Code Context

## Project Overview

[YOUR_PROJECT_NAME] is a [brief one-sentence description, e.g., "B2B SaaS platform for managing
construction project timelines"].

**Stack:** [YOUR_STACK — e.g., Node.js 20 / Express 5 / TypeScript 5.4 / React 18 / PostgreSQL 15 / Redis 7]

**Architecture:** Monorepo with two main packages:
- `packages/api` — REST API (Express), business logic, database access via Prisma
- `packages/web` — React SPA (Vite), Tailwind CSS, React Query for server state

**Deployment:** [e.g., AWS ECS (API) + CloudFront/S3 (web) via GitHub Actions]

---

## Stack and Conventions

**TypeScript:** Strict mode is on (`"strict": true`). No `any` types — use `unknown` and narrow,
or define a proper interface. Generics over type assertions.

**Naming:**
- Files: `kebab-case.ts` (e.g., `user-profile.service.ts`)
- Classes/Types/Interfaces: `PascalCase`
- Functions/variables: `camelCase`
- Constants: `UPPER_SNAKE_CASE`
- Database columns: `snake_case` (Prisma maps these automatically)

**File structure (API):**
```
src/
  routes/        # Express routers — thin, delegate to services
  services/      # Business logic — one class per domain entity
  repositories/  # Database access — no raw SQL outside this layer
  middleware/    # Auth, validation, error handling
  types/         # Shared TypeScript types and Zod schemas
```

**Imports:** Use path aliases (`@api/services/user`) not relative `../../` chains.
**Error handling:** All async route handlers wrapped in `asyncHandler`. Never swallow errors silently.
**Logging:** Use the shared `logger` from `@api/lib/logger` (Winston). Never use `console.log` in production code.

---

## Testing Requirements

- **Always run tests before finishing a task.** Command: `pnpm test` (runs both packages).
- Unit tests live next to the file they test: `user.service.test.ts` alongside `user.service.ts`.
- Integration tests live in `tests/integration/`.
- **Coverage thresholds** (enforced in CI): Statements 80%, Branches 75%, Functions 80%.
- New service methods must have at least one unit test covering the happy path and one error case.
- Use `vitest` for both packages. Do not introduce `jest`.
- Mock external services (Stripe, SendGrid, etc.) — never hit real APIs in tests.
- Snapshot tests are allowed only for UI components, not for API response shapes.

---

## Off-Limits Patterns

**NEVER do these without explicit human approval in the chat:**

1. **No `any` type.** If you're tempted, use `unknown` or define the type properly.
2. **No secrets or credentials in code.** All secrets via environment variables. If you see a hardcoded key, flag it immediately.
3. **No direct modifications to migration files** in `prisma/migrations/`. Create a new migration instead.
4. **No `git push` to `main` or `production`.** Branch and PR only.
5. **No installing new npm packages** without listing them and getting approval first.
6. **No bypassing the auth middleware** (`requireAuth`, `requireRole`) on new routes.
7. **No raw SQL** outside of the `repositories/` layer.
8. **No `console.log`** in committed code — use the logger.
9. **No disabling TypeScript errors** with `@ts-ignore` or `@ts-expect-error` without a comment explaining why.
10. **No modifying `.env.example`** to add real values — it's committed to the repo.

---

## Git Workflow

**Branch naming:** `[type]/[TICKET_NUMBER]-short-description`
- Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`
- Example: `feat/PROJ-412-user-export-csv`

**Commit message format** (Conventional Commits):
```
type(scope): short description under 72 chars

Optional body explaining the why, not the what.

Refs: PROJ-412
```

**Rules:**
- Never commit directly to `main` or `develop`.
- Every PR requires at least one approval before merge.
- Squash merge only (GitHub enforces this).
- Delete branch after merge.

---

## Architecture Constraints

These are decided. Do not re-litigate them or introduce alternatives without an ADR.

- **ORM:** Prisma only. No raw SQL outside repositories, no Knex, no TypeORM.
- **API style:** REST. No GraphQL endpoints.
- **Auth:** JWT (short-lived access tokens) + refresh token rotation. No sessions/cookies for API.
- **Frontend state:** React Query for server state, Zustand for client-only UI state. No Redux.
- **CSS:** Tailwind utility classes. No CSS-in-JS (no styled-components, no Emotion).
- **Queue:** BullMQ backed by Redis for async jobs. No in-process queues.
- **Dates:** Always store UTC in the database. Convert to user timezone only at the presentation layer.

---

## Corrections Log

When Claude makes a repeated mistake, add an entry here so it doesn't happen again.

**Format:**
```
[DATE] [CATEGORY] — What was wrong / What to do instead
```

**Log:**
```
[YYYY-MM-DD] TYPES — Used `any` for Express req.body. Use the typed Zod-parsed body instead:
  const body = req.body as z.infer<typeof CreateUserSchema>

[YYYY-MM-DD] TESTING — Imported vitest from wrong path. Use `import { describe, it, expect } from 'vitest'`
```
