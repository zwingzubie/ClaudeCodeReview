## AI-Assisted Security Review Checklist

**Related to:** [Security Overview](00-overview.md) — Security Area 6 · [Governance: Review Policies](../Governance/01-review-policies.md)[^a] · [QA & Testing: Test Session Design](../QA%20%26%20Testing/01-test-session-design.md)[^b] · [Security: Threat Modeling](01-threat-modeling.md)[^c] · [Governance: Compliance and Audit](../Governance/06-compliance-and-audit.md)[^d]

---

## Overview

Ad-hoc security review is not zero security review — it is security review with inconsistent coverage. An experienced engineer performing ad-hoc review may reliably catch SQL injection and XSS in categories they have been burned by before, while systematically missing authentication logic errors or cryptographic implementation weaknesses that have never appeared in their personal incident history. Ad-hoc review reflects the reviewer's history, not the full attack surface. A checklist ensures that every review covers the same categories in the same order, regardless of which reviewer is performing it or which category they find personally salient.[^1]

For AI-generated code, the inconsistency problem is more acute. AI introduces vulnerability patterns that are novel to the team's experience — patterns borrowed from training data that the team has not personally encountered — and that are presented in code that looks idiomatic and plausible. A reviewer who does not have the specific pattern in their mental security model may not recognize it as a vulnerability. A checklist that includes the category forces the reviewer to ask the question even if they would not have thought to ask it unprompted. The checklist is not a substitute for expertise — it is a structure that ensures expertise is applied consistently rather than applied only to the categories the expert happens to be thinking about.[^2]

A standardized checklist also creates a documentation artifact. A review process that leaves no record of what was checked provides no evidence for a post-incident review or compliance audit. A checklist with checked boxes and a reviewer signature is evidence that the review occurred, what it covered, and who performed it. For a team without a dedicated security engineer, this documentation artifact may be the primary record of the security review process that a customer audit or compliance review will examine.

---

## Section 1: Why a Standardized Checklist Beats Ad-Hoc Review for AI-Generated Code

**Description:** The research on checklist effectiveness in complex technical domains is consistent: checklists reduce error rates by enforcing systematic coverage of a category list that is too long and too low-salience to reliably reconstruct from memory under time pressure. Atul Gawande's 2009 work on surgical checklists is the most cited example, but the mechanism applies directly to security code review: the failure mode in both cases is not ignorance but distraction — the reviewer knows what to check but skips a category because their attention is on a more salient issue in the specific case at hand.[^3]

AI-generated code has a specific property that makes this failure mode more common: it is syntactically idiomatic and functionally plausible. A human reviewer reading AI-generated authentication code that looks like well-structured, convention-following code is cognitively primed to approve it — the surface characteristics signal correctness. The authorization logic error or the missing rate limiting that the code contains does not announce itself; the reviewer must specifically ask the authorization question to find it. A checklist that includes "authorization: does every endpoint verify that the authenticated user is authorized for the requested resource?" makes the reviewer ask the question instead of relying on the code's appearance to signal whether the question is worth asking.[^2]

**Recommended Practice:**
- Present the checklist data to the team: ad-hoc security review misses a different subset of categories for each reviewer; standardized checklist review misses categories that are absent from the checklist. The latter failure mode is correctable (update the checklist); the former is not (it reflects each reviewer's idiosyncratic attention pattern). This framing makes the case for the checklist more concretely than "we should be more consistent."[^1]
- Keep the checklist short enough to complete in under 15 minutes for a typical PR. A checklist with 50 items will not be completed; it will be skimmed or skipped. Each category should have 3–5 specific checks; the total checklist should have 6–8 categories. Ruthlessly trim items that do not reflect actual AI vulnerability patterns in the team's stack.
- Update the checklist when the team's vulnerability log produces a finding that the checklist would not have caught. The checklist is a living document, not a one-time artifact. A finding that slips through the checklist is evidence that the checklist needs a new item, not just that the reviewer should have been more careful.
- Use the checklist as a team calibration tool: periodically conduct a review exercise where engineers independently apply the checklist to the same AI-generated code sample and compare results. Differences in checklist application reveal where the items need clarification and where engineers need additional training.

---

## Section 2: Checklist Items Organized by Category

**Description:** The checklist categories for AI-generated code review map directly to the OWASP Top 10 and to the vulnerability categories where AI-generated code has documented higher failure rates: input handling, authentication, authorization, cryptography, injection, dependency additions, and secrets. These categories cover the most common attack vectors and the specific weakness areas that Sonar's 2026 research and Veracode's Spring 2026 analysis identified as concentrated in AI-generated output. The category organization is not arbitrary — it reflects the structure of the team's SAST configuration and the security review categories in the vulnerability log, ensuring that checklist items correspond to trackable finding categories.[^5]

The checklist items are written as questions rather than assertions because questions require an answer — the reviewer must determine whether the condition is met, not simply acknowledge that the category exists. "Is user input sanitized before database queries?" is a checklist item that requires the reviewer to look at the code and answer yes or no. "Sanitize user input" is a reminder that may be read without triggering a specific review action. The question format is not stylistic preference; it is the behavioral design choice that makes checklists effective.

**Recommended Practice:**
- Apply the following checklist to all AI-generated code in security-critical paths:

 **Input Validation and Sanitization**
 - Is all user input validated against an expected type and range before processing?
 - Are string inputs sanitized or rejected before being used in database queries, file paths, or command execution?
 - Are uploaded files validated for type, size, and content before being stored or processed?

 **Authentication and Session Management**
 - Does AI-generated authentication code use the team's standard authentication library rather than a custom implementation?
 - Are session tokens generated with sufficient entropy and invalidated correctly on logout and timeout?
 - Are authentication errors handled consistently without leaking information about valid usernames?

 **Authorization and Access Control**
 - Does every endpoint that accesses a resource verify that the authenticated user is authorized for that specific resource?
 - Are authorization checks applied at the data access layer, not only at the route or controller layer?
 - Does the code prevent horizontal privilege escalation (accessing another user's data with a valid but wrong ID)?

 **Cryptographic Operations**
 - Does AI-generated cryptographic code use the team's approved cryptographic library rather than a custom implementation?
 - Are encryption keys sourced from environment variables, not hardcoded or derived from predictable values?
 - Are password hashing functions using approved algorithms (bcrypt, Argon2) rather than MD5, SHA-1, or unsalted hashes?

 **Injection Prevention**
 - Are all database queries using parameterized queries or ORM parameter binding, with no string concatenation?
 - Are shell commands constructed without user-supplied values, or using a safe API that prevents command injection?
 - Are template expressions escaped correctly for the output context (HTML, SQL, LDAP)?

 **Dependency Additions**
 - Did this PR add any new dependencies? If so, were they reviewed against the four-criteria checklist?
 - Are new dependencies using versions without known CVEs in the team's vulnerability database?

 **Secrets and Credentials**
 - Are all credentials referenced via environment variables, with no string literals containing keys, passwords, or tokens?
 - Does the PR diff include any high-entropy strings that could be actual credentials inadvertently committed?[^5]

---

## Section 3: Using the Checklist as a Claude Code Slash Command

**Description:** A checklist that lives in a document is a checklist that requires engineers to navigate to that document, remember it exists, and copy it into their review process. A checklist implemented as a Claude Code slash command is a checklist that is available immediately, in the development environment where the review is happening, with a single command invocation. The slash command format also allows the checklist to be parameterized by context: the `/security-review` command can accept the names of files or directories to focus on, and can generate a review session that applies the checklist specifically to the AI-generated code in the current working directory.[^6]

Claude Code custom commands are defined in the `.claude/commands/` directory as Markdown files containing the prompt to inject. The security review command prompt should include the full checklist, an instruction to review the specified files against each checklist item, and a request to produce a structured output with findings per category rather than a narrative. Structured output makes the review result usable as a PR comment artifact and as a vulnerability log entry, rather than a freeform text that must be interpreted.

**Recommended Practice:**
- Create `.claude/commands/security-review.md` with a prompt that: (1) states the review objective and scope, (2) provides the full checklist organized by category, (3) instructs Claude to review each category against the specified files and report findings as a structured list with file references, line numbers, and severity assessments, and (4) requests a summary table of categories reviewed, findings found, and recommended actions.[^6]
- Add `/security-review` to the team's standard AI-primary PR workflow documentation: before requesting human review, the PR author runs `/security-review` on the AI-generated files and includes the output in the PR description. This creates a self-review artifact that human reviewers can use as a starting point rather than reviewing from scratch.
- Update the slash command prompt when the team's checklist is updated. The command file is version-controlled alongside the codebase; the git history of `.claude/commands/security-review.md` is the history of the team's security review standards evolution.
- Do not rely on the slash command output as a substitute for human security review. The command applies AI to security review of AI-generated code — the result is a useful pre-screening artifact, not a security certification. Human review of the command output is still required for PRs in security-critical paths.

---

## Section 4: Integrating Checklist Results into PR Comments via CI

**Description:** A security review that produces findings that do not appear in the PR review record is a security review that may not produce action. PR comments are the canonical record of what was reviewed, what was found, and what was decided before a PR was merged. A CI step that automatically generates a security review summary as a PR comment ensures that every PR in security-critical paths has a visible record of the security review, regardless of whether the engineer ran the slash command manually or whether the reviewer noted their security review process in a comment.[^7]

The CI integration architecture is straightforward: a GitHub Actions workflow that triggers on PR creation and update, runs a lightweight SAST scan on changed files, and posts the results as a PR comment using the `actions/github-script` action or a dedicated PR review tool. The comment format should include: checklist categories reviewed, automated findings per category (with file and line references), a recommendation for which findings require human review before merge, and a reviewer action prompt that asks the human reviewer to confirm they have reviewed each finding.[^7]

**Recommended Practice:**
- Configure a GitHub Actions workflow (`.github/workflows/security-review-comment.yml`) that runs on `pull_request` events, executes Semgrep against changed files, and posts a formatted security review summary as a PR comment. Use the `semgrep --json` output format to produce structured findings that can be formatted into a readable comment.[^7]
- Format the PR comment using a checklist-organized structure rather than a flat finding list: group findings by checklist category so that the reviewer can immediately see which categories have findings and which are clean. A reviewer who sees "Authorization: 1 finding, Injection: 0 findings, Secrets: 0 findings" can triage their review attention more efficiently than one who reads a list of findings without category context.
- Configure the CI workflow to block PR merge when the comment identifies Critical or High findings, using a required status check on the workflow. A PR that cannot be merged until critical findings are addressed is a PR where the security review process has operational effect rather than advisory effect.
- Archive the security review PR comments in the team's vulnerability log for any PR where findings were identified. The comment record is the documentation artifact that links the vulnerability finding, the review process, and the resolution decision — the three elements that post-incident and compliance reviews will look for.

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Checklist Adoption | Draft and distribute security review checklist; conduct first team calibration session | Architect |
| Slash Command | Create `.claude/commands/security-review.md`; document in PR workflow | Backend lead |
| CI PR Comment | Configure GitHub Actions security review comment workflow | Backend lead |
| Checklist Maintenance | Add checklist update to post-incident retrospective process | Architect |

---

[^1]: OWASP — "OWASP Code Review Guide v2," OWASP Foundation, 2017. https://owasp.org/www-project-code-review-guide/
 The argument for systematic over ad-hoc security review; coverage consistency across reviewers; the checklist as the mechanism for standardizing security review in the absence of dedicated security engineering resources.

[^2]: Veracode — "Spring 2026 GenAI Code Security Update: Despite Claims, AI Models Are Still Failing Security," March 24, 2026. https://www.veracode.com/blog/spring-2026-genai-code-security/
 AI-generated code vulnerability categories; the plausibility problem for reviewer cognitive priming; the case for checklist-enforced category coverage for AI-generated code review.

[^3]: Atul Gawande — *The Checklist Manifesto: How to Get Things Right*, Metropolitan Books, 2009.
 The systematic evidence for checklist effectiveness in complex technical domains; the mechanism by which checklists prevent errors of distraction rather than errors of ignorance; application to technical review processes.

[^5]: OWASP — "OWASP Top 10 2021," OWASP Foundation. https://owasp.org/Top10/
 Security review category taxonomy: A01 Broken Access Control, A02 Cryptographic Failures, A03 Injection, A07 Identification and Authentication Failures; the OWASP Top 10 as the foundation for checklist category selection.

[^6]: Anthropic — "Claude Code: Custom Commands," Claude Code Documentation, 2026. https://code.claude.com/docs/en/custom-commands
 Slash command implementation via `.claude/commands/` directory; command prompt structure for structured output generation; `/security-review` command design patterns.

[^7]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
 CI integration for automated PR security comment generation; Semgrep JSON output formatting for PR comments; required status check configuration for blocking merge on critical findings.

[^a]: [Governance: Review Policies](../Governance/01-review-policies.md) — the security review checklist is a required artifact within the review policy workflow; policy mandates when the checklist is used and what completion means.

[^b]: [QA & Testing: Test Session Design](../QA%20%26%20Testing/01-test-session-design.md) — QA test session design includes security-specific test cases; the checklist and test session design are parallel review artifacts covering the same PR from different perspectives.

[^c]: [Security: Threat Modeling](01-threat-modeling.md) — threat model outputs determine which checklist sections are most critical for a given component; the threat model informs how the checklist is applied.

[^d]: [Governance: Compliance and Audit](../Governance/06-compliance-and-audit.md) — completed checklists are compliance artifacts; audit requirements reference checklist completion rates as evidence of systematic security review.
