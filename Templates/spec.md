# Spec: [ONE_SENTENCE_DESCRIPTION_OF_THE_FEATURE]

**Ticket:** [TICKET_NUMBER] — [TICKET_TITLE]
**Date:** [YYYY-MM-DD]
**Author:** [YOUR_NAME]
**Branch:** `feat/[TICKET_NUMBER]-[short-slug]`

---

## Objective

[One sentence. What should exist when this task is done that doesn't exist now?]

Example: "Add a CSV export endpoint for user activity reports that product managers can trigger
from the admin dashboard."

---

## Context

[Background Claude needs to understand the problem. Link to relevant resources.]

- **Linear ticket:** [TICKET_NUMBER] — [link if using MCP, or paste key details here]
- **Related PRs / prior work:** [link or PR number]
- **Design doc / PRD:** [Google Drive link or inline summary]
- **Why now:** [Brief reason this is being built — sprint goal, customer request, etc.]

**Architectural constraints to keep in mind:**
- [e.g., "This touches the billing service — do not modify Stripe webhook handlers"]
- [e.g., "The report query must use the read replica connection (`db.replica`), not the primary"]

---

## Requirements

Numbered, written as acceptance criteria. Claude should treat these as the definition of done.

1. `GET /api/v1/reports/users/export` returns a CSV file with columns: `id`, `email`, `created_at`, `last_active_at`, `plan`.
2. The endpoint requires the `admin` role (use `requireRole('admin')` middleware).
3. Date range is accepted as query params `from` and `to` (ISO 8601). Both are required; return 400 if missing or invalid.
4. Rows are sorted by `created_at` descending.
5. Response includes header `Content-Disposition: attachment; filename="users-[date].csv"`.
6. A unit test covers: happy path, missing params (400), invalid date format (400), empty result set.
7. An integration test hits the real DB (test database) and verifies the CSV structure.

---

## Out of Scope

Be explicit. Claude should not build these even if they seem like natural extensions.

- No streaming / chunked response — the dataset is small enough for a single response.
- No email delivery of the report — that's a separate ticket ([TICKET_NUMBER]).
- No frontend changes in this session — the UI will be addressed in [TICKET_NUMBER].
- No pagination.

---

## Files to Modify

List every file Claude should touch. This bounds scope and prevents accidental changes.

```
packages/api/src/routes/reports.router.ts          # Add new route
packages/api/src/services/user-report.service.ts   # CREATE — new file
packages/api/src/repositories/user.repository.ts   # Add getUsersForExport() query
packages/api/src/types/report.types.ts             # Add ExportQueryParams type
packages/api/src/routes/reports.router.test.ts     # Unit tests for the route
tests/integration/reports.export.test.ts           # Integration test
```

**Do not modify:** anything in `prisma/migrations/`, `packages/web/`, or `.github/`.

---

## Verification

Commands to run to confirm the task is complete. Claude should run these before declaring done.

```bash
# Type check (must pass with 0 errors)
pnpm tsc --noEmit

# Unit tests (must pass, coverage thresholds must hold)
pnpm --filter api test --coverage

# Integration tests
pnpm --filter api test:integration

# Manual smoke test (requires local server running)
curl -s -o /tmp/export.csv -D - \
  "http://localhost:3000/api/v1/reports/users/export?from=2024-01-01&to=2024-12-31" \
  -H "Authorization: Bearer [ADMIN_TEST_TOKEN]"
head -3 /tmp/export.csv   # Should show CSV headers then data rows
```

---

## Notes for Claude

[Anything specific Claude needs to know that isn't covered above.]

- The `user-report.service.ts` file should follow the pattern in `user.service.ts` — constructor injection of the repository, no direct Prisma calls.
- Use the `toCsv()` utility in `@api/lib/csv` — don't write CSV serialization from scratch.
- The `from`/`to` validation should use the existing `isValidISODate()` helper in `@api/lib/validation`.
- If you're unsure about the read-replica pattern, read `db.repository.ts` before writing the query.
- Ask before installing any new dependencies.
