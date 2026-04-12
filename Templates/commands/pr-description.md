# PR Description

You are generating a pull request description for the current branch. Your output should be
ready to paste directly into GitHub with minimal editing.

---

## Step 1: Gather Information

Run the following commands to understand what changed:

```bash
# What branch is this?
git branch --show-current

# What commits are in this branch that aren't in main?
git log main...HEAD --oneline

# What files changed?
git diff main...HEAD --name-only

# Full diff (for understanding the changes)
git diff main...HEAD
```

Also check for a spec file:
- Look in `./specs/` for a file matching the branch name or ticket number
- If found, read it — the spec's Objective and Requirements are useful for the summary

---

## Step 2: Analyze the Changes

For each changed file, identify:
1. **Why it changed** — what feature, fix, or refactor drove this change
2. **What specifically changed** — new functions, modified logic, added tests
3. **Risk level** — is this a trivial change or does it touch critical paths (auth, payments, data migrations)?

Group related changes together into logical change items.

---

## Step 3: Write the PR Description

Output the description in the following format. Use the actual content you found — do not use
placeholder text in your output.

```markdown
## Summary

[1-2 paragraph explanation of what this PR does and why. Write for a reviewer who has context
on the codebase but hasn't been following this ticket. Mention the ticket number.]

**Ticket:** [TICKET_NUMBER if found in branch name or spec, otherwise omit]

---

## Changes

[Bullet list. Each bullet = one logical change. Be specific about what changed and why.
Example: "- `user-report.service.ts` (new): Implements CSV generation for user activity exports, 
  using the `toCsv()` utility and the read replica connection."]

---

## Testing

[Describe what tests were added or modified. Mention test commands.
Example: "Added 6 unit tests covering happy path, missing params (400), invalid date format (400),
and empty result set. Integration test verifies CSV structure against test database."]

- [ ] `pnpm test` passes
- [ ] `pnpm tsc --noEmit` passes
- [ ] Coverage thresholds met

---

## AI Involvement

[Fill in honestly based on the session that produced this code.]

**Approximate % AI-generated:** [your estimate]
**Sessions:** [brief description of what Claude did in this branch]

---

## Reviewer Guidance

[Be specific about where reviewers should focus energy. Flag anything you're uncertain about.
Example: "Please review the SQL query in user.repository.ts carefully — I want to confirm
the index on (created_at, deleted_at) is being used for this query shape."]

**Focus here:**
- [area 1]
- [area 2]

**Can skim:**
- [area — e.g., "Test file follows established pattern"]
```

---

## Step 4: Call Out Any Concerns

After the PR description, add a separate section (not for the PR body, for the engineer's eyes):

```
--- NOTES FOR THE ENGINEER ---
[Any issues you noticed while reading the diff that should be addressed before opening the PR:
 - Missing tests
 - Potential bugs
 - Off-limits patterns violated (from CLAUDE.md)
 - Large or risky changes that warrant extra review
 If everything looks clean, say so.]
```
