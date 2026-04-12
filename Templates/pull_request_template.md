## Summary

<!-- One paragraph: what problem does this PR solve and how? -->

**Ticket:** [TICKET_NUMBER] — [TICKET_TITLE]

---

## Changes

<!-- Bullet list of what changed. Be specific about files/modules touched. -->

- 
- 
- 

---

## Testing

<!-- What tests were added or modified? Did all tests pass? -->

- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] `pnpm test` passes locally with no failures
- [ ] Coverage thresholds still met (run `pnpm test --coverage` to verify)

**Test commands to verify:**
```bash
pnpm test
pnpm tsc --noEmit
```

---

## AI Involvement

<!-- Be honest. This helps reviewers calibrate how much scrutiny to apply. -->

**Approximate % of code AI-generated:** <!-- e.g., 80% -->

**Claude Code sessions used:**
- Session 1: <!-- e.g., "Scaffolded service and route, wrote unit tests" -->
- Session 2: <!-- e.g., "Fixed type errors flagged by tsc, added missing error case" -->

**How you used Claude:**
- [ ] Claude wrote first draft, I reviewed and edited
- [ ] Claude assisted with specific sections (describe below)
- [ ] I wrote code, Claude was used only for review/Q&A
- [ ] Other: <!-- describe -->

<!-- Describe any specific sections where Claude's output surprised you or required significant correction: -->

---

## Comprehension Declaration

**I, [YOUR_NAME], attest that:**

- [ ] I have read every line of code in this PR and understand what it does
- [ ] I can explain the logic of any function in this PR without referring to notes
- [ ] I understand the edge cases covered by the tests and why they matter
- [ ] I am confident this code is correct and production-ready

<!-- If any checkbox is unchecked, explain why and what additional review is needed: -->

---

## Security Check

- [ ] SAST scan run (semgrep or `npm audit`) — no HIGH/CRITICAL findings
- [ ] No secrets, API keys, or credentials in code or test fixtures
- [ ] New dependencies reviewed (`npm audit` clean, licenses checked)
- [ ] Input validation present on all new endpoints/functions that accept external data
- [ ] Auth/authz middleware applied to all new routes

**Scan output summary:** <!-- paste last few lines of semgrep/audit output, or "clean" -->

---

## Reviewer Guidance

<!-- Tell reviewers exactly where to focus. Don't make them hunt. -->

**Please pay close attention to:**
- <!-- e.g., "The SQL query in user.repository.ts — I'm not 100% sure the index is being used" -->
- <!-- e.g., "The error handling in the export service — make sure all failure modes are covered" -->

**You can skim:**
- <!-- e.g., "The test file — it follows the established pattern exactly" -->

**Questions for reviewers:**
- <!-- Any open questions you want feedback on -->
