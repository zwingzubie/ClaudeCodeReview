# Feature Scaffold

You are scaffolding a new feature. Your job is to create a correct, minimal structure that the
engineer can build on — not to implement the full feature logic.

**Stop and ask before making any file changes.** The approval step below is mandatory.

---

## Step 1: Read the Spec

Look for a spec file in the following locations (in order):
1. A file path mentioned in the current conversation
2. `./specs/` — find the most recently modified `.md` file
3. Ask the engineer: "Which spec file should I use for this scaffold?"

Read the spec completely. Identify:
- The feature's objective
- The files listed in "Files to Modify"
- Any architectural constraints mentioned

---

## Step 2: Plan — List Files Before Touching Anything

Before creating or modifying a single file, output a plan in this exact format:

```
Scaffold Plan for: [FEATURE NAME from spec]

FILES TO CREATE:
  [relative/path/to/file.ts]     — [one-line description of what it contains]
  [relative/path/to/file.test.ts] — [one-line description]

FILES TO MODIFY:
  [relative/path/to/existing.ts] — [what will be added/changed]

NAMING CONVENTIONS APPLIED:
  - [Note any specific convention being followed, e.g., "service files use kebab-case"]

DEPENDENCIES NEEDED:
  - [Any new npm packages required, or "none"]

QUESTIONS BEFORE PROCEEDING:
  - [Any ambiguity in the spec that needs clarification before you scaffold]
```

**Wait for the engineer to respond with "proceed" or corrections before continuing.**
Do not create any files until you receive explicit approval.

---

## Step 3: Scaffold Files

Once approved, create each file in the plan. Follow these rules:

**For implementation files:**
- Create the class/function signature with correct TypeScript types
- Add a `// TODO: implement` comment in the body — do not write business logic
- Import only what is needed for the signature to type-check
- Follow the naming conventions in `CLAUDE.md` exactly

**For test files:**
- Use `vitest` (`import { describe, it, expect, vi } from 'vitest'`)
- Create `describe` blocks and stub `it` cases for: happy path, each error case from the spec, edge cases
- Mark stub tests with `it.todo('...')` so they appear in test output as pending
- Add one real test for the most critical happy path if the implementation is clear from the spec

**For route files:**
- Register the route with the correct HTTP method and path from the spec
- Apply `requireAuth` and `requireRole` middleware per the spec's auth requirements
- Wrap handler in `asyncHandler`

---

## Step 4: Confirm

After creating all files, output:
```
Scaffold complete.

Created:
  [list of files created with relative paths]

Modified:
  [list of files modified]

Next steps for the engineer:
  1. Run `pnpm tsc --noEmit` to verify types compile
  2. Run `pnpm test` to see pending tests
  3. Implement the TODOs in [list the implementation files]
  4. Fill in the stub tests in [list the test files]
```
