## Task-Specific Prompt Patterns: Matching Prompt Structure to Task Type

**Related to:** [Prompting Overview](00-overview.md) — Pattern 2 · [Workflows: Task Decomposition](../Workflows/02-task-decomposition.md)[^a] · [Prompting: Prompt Architecture](01-prompt-architecture.md)[^b] · [Tooling: Custom Skills](../Tooling & Configuration/04-custom-skills.md)[^c]

---

## Overview

Different task types require structurally different prompts. A feature scaffolding prompt needs upfront architectural context before any generation; a refactoring prompt needs explicit preservation constraints before any change directives; a security review prompt needs a vulnerability class enumeration before any code review; a test generation prompt needs the function under test plus its dependencies before any test scaffolding. Using a single generic prompt structure for all task types systematically degrades output quality across all of them — not because the prompt is bad but because it is calibrated to no task type in particular.[^1]

This memo documents prompt patterns for the five task types that represent the majority of AI-assisted development work: feature scaffolding, refactoring, test generation, security review, and documentation. For each, it specifies the structural components that matter most, the constraints that prevent the most common failure modes, and the verification approach appropriate to the output type. These patterns are the foundation for the team's custom command library (see Prompting Overview — Pattern 6).

---

## Section 1: Feature Scaffolding Prompts

**Description:** Feature scaffolding is the task type where architectural context matters most. When Claude generates a new feature from scratch, it makes dozens of architectural micro-decisions: how to structure the module, what naming to use, which patterns to apply, how to handle errors, where to put tests. Without explicit constraint, each of these decisions is made by matching against training data patterns — which may or may not match the team's codebase. The more complex the feature, the more decisions are made this way, and the more the resulting code diverges from team conventions.[^2]

The most effective scaffolding prompts follow a consistent three-phase structure. First, an explicit Explore instruction: "Read [list of relevant existing files] to understand our existing patterns for [relevant domain]." This grounds Claude in the actual codebase before any generation. Second, a planning gate: "Produce an implementation plan for [feature]. Do not write code yet." This enables plan review before implementation begins. Third, the implementation instruction with explicit file and pattern references: "Implement the plan, following the pattern in [reference file] for [specific aspect]."[^3]

**Recommended Practice:**
- Always begin a scaffolding prompt with the Explore phase, even when the relevant files seem obvious. "Read `src/api/users.ts` and `src/api/products.ts` to understand our handler and routing patterns before implementing the new endpoint" produces more consistent output than assuming Claude will infer the pattern from context.[^3]
- Name the specific existing files whose patterns should be followed, rather than describing the pattern abstractly. "Follow the authentication pattern in `src/middleware/auth.ts`" is more precise than "follow our standard authentication pattern." Concrete file references override training data defaults.[^2]
- For features with multiple components (frontend + backend + tests), decompose into separate prompts for each component rather than asking for all three simultaneously. Each component benefits from focused context; a prompt asking for the complete feature tends to produce each component more shallowly than a focused prompt for each.[^4]
- Set a complexity threshold: features requiring more than five new files should be broken into spec.md tasks (see Workflows — Context Engineering) rather than addressed in a single scaffolding prompt. Above this threshold, a single prompt cannot adequately specify all the architectural decisions involved.[^5]

---

## Section 2: Refactoring Prompts

**Description:** Refactoring is the task type where constraint specification matters most. The failure mode of a poorly structured refactoring prompt is not that the refactoring fails — it is that the refactoring succeeds at what was requested while also changing things that were not requested. An engineer who asks Claude to "simplify the error handling in the payment module" may receive back a payment module with simplified error handling and a modified API surface, renamed methods, and a changed return type. The refactoring succeeded; the unasked-for changes broke three downstream consumers.[^6]

The preservation constraint — explicitly specifying what must not change — is the most important structural element of a refactoring prompt. It is also the most commonly omitted one, because engineers tend to focus on what should change rather than what should stay the same. For refactoring specifically, the preservation constraint deserves to come before the change directive: "Without changing the public API or return types, simplify the error handling in the payment module."[^7]

**Recommended Practice:**
- Lead every refactoring prompt with preservation constraints: "Do not change [function signatures / return types / API surface / test behavior / database schema]." Write these before describing what should change — the ordering reinforces their priority.[^6]
- Specify the refactoring scope file-by-file for multi-file refactors: "Refactor the error handling in `src/payment/processor.ts` only. Do not modify the callers in `src/api/orders.ts` or `src/api/subscriptions.ts`." Unbounded multi-file refactoring prompts tend to produce changes far outside the intended scope.[^2]
- Include a test run as the verification component for every refactoring prompt: "After refactoring, run the existing tests for this module and fix any failures. Do not add or remove tests — existing test behavior should be unchanged." This validates behavior preservation rather than assuming the refactored code is equivalent.[^7]
- For significant refactors, add an architectural intent statement: "The goal is to replace nested callbacks with async/await. The resulting code should be functionally identical, only structurally simpler." This gives Claude an evaluation criterion it can apply to its own output — a form of self-verification before submitting for review.[^3]

---

## Section 3: Test Generation Prompts

**Description:** Test generation is the task type where dependency context matters most. A test that does not accurately reflect how the function being tested actually behaves — because it was generated without access to the function's dependencies, external contracts, or error modes — produces a false sense of coverage. It passes without catching the bugs it was meant to catch. The most common root cause is a test generation prompt that provides the function signature without its dependencies and integration context.[^8]

Test generation also has a specific verification challenge: the test was generated by the same session that might have generated the implementation. A test that was designed to validate AI-generated code, generated in the same session as that code, may be inadvertently structured to pass the specific (possibly incorrect) implementation rather than to validate the intended behavior. This is a version of the writer/reviewer problem applied to test generation.[^9]

**Recommended Practice:**
- Provide the function under test, its dependencies, and at least two example usage patterns in the prompt: "Generate tests for `processPayment`. Here is the function: [code]. It calls `StripeClient.charge` (interface: [interface]) and `OrderRepository.updateStatus` (interface: [interface]). Tests should cover success, charge failure, and database failure."[^8]
- Specify the test framework and any team conventions in the prompt: "Write Jest tests. Follow the pattern in `tests/api/users.test.ts` for mock setup and assertion style." Do not assume Claude will infer the team's testing conventions from file names alone.[^2]
- Use a separate session for test generation when testing AI-generated code: a fresh-context session asked to generate tests for a function — without seeing the implementation — is more likely to test the specified behavior rather than the specific implementation. This is the test generation equivalent of the writer/reviewer pattern.[^9]
- Include edge case prompting explicitly: "Include tests for null inputs, empty arrays, network timeout scenarios, and concurrent calls." Claude tends to generate happy-path tests by default; edge cases require explicit request.[^8]

---

## Section 4: Security Review Prompts

**Description:** Security review is the task type where vulnerability scope specification matters most. A prompt that asks Claude to "review for security issues" without specifying the vulnerability classes produces a review calibrated to the most common issues in Claude's training data — which may not match the specific risk profile of the code being reviewed. Veracode's Spring 2026 analysis found that AI models consistently miss certain vulnerability classes while over-reporting others; a well-scoped security review prompt guides attention to the classes most relevant to the code type.[^10]

The most effective security review prompts specify: the vulnerability classes to focus on (injection, authentication, secrets management, IDOR, deserialization), the code context (what the code does, what external systems it interacts with, what data it handles), and the output format (severity classification, specific line references, recommended fixes rather than general guidance).[^11]

**Recommended Practice:**
- Enumerate the vulnerability classes explicitly in every security review prompt. For API endpoint code: "Review for SQL injection, authentication bypass, parameter pollution, unauthorized data access, and secrets in code." For authentication code: "Review for session fixation, token validation bypass, brute force susceptibility, and insecure credential storage."[^10]
- Reference the team's threat model or security invariants in the review prompt: "We use JWT with HS256 signing. Review for token validation gaps, algorithm substitution vulnerabilities, and expiration handling edge cases." This grounds the review in the team's specific security architecture rather than generic best practices.[^11]
- Request output with severity classification and specific line references: "For each finding, classify as Critical/High/Medium/Low, cite the specific line numbers, and suggest a concrete fix." Generic findings ("improve error handling") are actionable; non-specific findings ("consider security hardening") are not.[^2]
- Run security reviews in a separate session from implementation, with access to CLAUDE.md and the team's threat model but without access to the implementation session's context. A fresh-context review session cannot rationalize the implementation choices the implementing session made — it is forced to evaluate the code as it appears.[^9]

---

## Section 5: Documentation Prompts

**Description:** Documentation generation is the task type where audience specification matters most. Claude's default documentation voice is generic — technical, comprehensive, API-reference style. Most real documentation needs are more specific: onboarding documentation for engineers who are new to a module, operational runbooks for on-call engineers dealing with an incident, API reference for frontend engineers consuming a backend endpoint, architecture decision records for future engineers making related decisions. Each audience requires different content emphasis, depth, and language.[^12]

Documentation prompts that do not specify the audience, purpose, and format produce documentation that is technically accurate but practically not useful — because it is calibrated to no specific reader. The engineer who uses the documentation is often not the engineer who prompted it, and the documentation that would help the actual reader is knowable in advance.[^7]

**Recommended Practice:**
- Begin every documentation prompt with audience and purpose: "Generate onboarding documentation for a new backend engineer who needs to understand how our authentication flow works. Assume familiarity with JWT but no familiarity with our specific implementation."[^12]
- Specify the documentation format explicitly: "Use ADR format (context / decision / consequences) for the architecture note." "Use a step-by-step numbered format for the runbook." "Use JSDoc format for the API reference." Format specification is the most reliable way to get documentation that fits into the team's existing documentation patterns.[^3]
- For operational documentation (runbooks, incident response guides), include the specific triggers and failure modes that motivated the documentation: "This runbook is for the payment processing timeout incident. Include: how to identify the issue, how to verify impact, escalation path if the fix does not work within 30 minutes."[^12]
- After generating documentation, run it through a comprehension test: ask a team member who did not write the documented code to execute the runbook or follow the onboarding guide. If they encounter confusion, that confusion is the documentation's failure mode — use it to improve the prompt for the next generation cycle.[^8]

---

## Summary of Recommended Practices

| Task Type | Key Structural Element | Immediate Action |
|---|---|---|
| Feature Scaffolding | Explicit Explore phase + named reference files | Add explicit Explore instruction to scaffold command |
| Refactoring | Preservation constraints before change directives | Update refactor command to lead with "do not change" |
| Test Generation | Dependency context + separate session for AI-generated code | Add dependency template to test command |
| Security Review | Vulnerability class enumeration + severity classification output | Add vulnerability list to security review command |
| Documentation | Audience specification + format directive | Add audience/purpose template to docs command |

---

[^1]: Phillip Carter — "How I Code With LLMs These Days," Honeycomb, March 2025. https://www.honeycomb.io/blog/how-i-code-with-llms-these-days
    Task suitability framework: different task types require structurally different prompts; the systematic quality degradation from applying a generic prompt structure to all task types.

[^2]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    Task-specific prompt guidelines: the Explore-Plan-Implement pattern for scaffolding, preservation constraints for refactoring, test specification requirements, and minimum context principles.

[^3]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    Concrete file references over abstract descriptions: why naming specific files produces more architecturally aligned output than describing patterns verbally; planning gates before implementation.

[^4]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    Feature decomposition by component type: why separate prompts for frontend, backend, and test components produce better results than unified feature prompts; the 15-minute waterfall principle.

[^5]: Artur Less — "Spec-Driven Development with Claude Code," Level Up Coding / Medium, March 2026. https://levelup.gitconnected.com/spec-driven-development-with-claude-code-1b08184965e3
    Complexity threshold for spec.md escalation: when single-prompt scaffolding is insufficient and the feature requires the full spec-driven development workflow.

[^6]: DEV Community — "AI Is Creating a New Kind of Tech Debt — And Nobody Is Talking About It," March 2026. https://dev.to/harsh2644/ai-is-creating-a-new-kind-of-tech-debt-and-nobody-is-talking-about-it-3pm6
    Refactoring failure modes: unasked-for changes as the primary source of AI-generated technical debt from refactoring sessions; preservation constraints as the structural countermeasure.

[^7]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    Refactoring prompt structure, test generation with dependency context, documentation audience specification, and the writer/reviewer pattern for test generation of AI-generated code.

[^8]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    Documentation prompt design: audience-specific documentation generation and the comprehension test as a quality validation mechanism for AI-generated documentation.

[^9]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
    Writer/reviewer pattern applied to test generation: fresh-context sessions for test generation of AI-generated code; security reviews in separate sessions without implementation context.

[^10]: Veracode — "Spring 2026 GenAI Code Security Update: Despite Claims, AI Models Are Still Failing Security," March 24, 2026. https://www.veracode.com/blog/spring-2026-genai-code-security/
    Security review prompt effectiveness: vulnerability class specification for targeted review; the evidence for AI's inconsistent coverage of different vulnerability classes without explicit guidance.

[^11]: Dark Reading — "AI-Generated Code Poses Security, Bloat Challenges," October 2025. https://www.darkreading.com/application-security/ai-generated-code-leading-expanded-technical-security-debt
    Security review prompt design: threat model reference, code context specification, and structured output format (severity + line reference + fix) for actionable security findings.

[^12]: Kyros — "The Vibe Coding Crisis: How AI-Generated Technical Debt Is Costing Companies Millions," March 2026. https://usekyros.ai/blog/vibe-coding-crisis-ai-technical-debt
    Documentation debt as a subset of AI-generated technical debt: how generic documentation fails the specific readers who need it, and audience-driven prompt design as the mitigation.

[^13]: Sabrina Ramonov — "CLAUDE CODE FULL COURSE," YouTube, February 17, 2025. https://www.youtube.com/watch?v=fYX6hHC9FhQ
    - Task-specific command library: the five foundational command types (scaffold, refactor, test, security, docs) and how each is structured to match its task type's requirements
    - Refactoring command walkthrough: live demonstration of a preservation-constraint-leading refactoring prompt and the observable difference from a generic refactoring request
    - Security review command: vulnerability class enumeration, threat model reference, and severity-classified output format in a reusable team command


[^a]: [Workflows: Task Decomposition](../Workflows/02-task-decomposition.md) — task decomposition determines what tasks are appropriate for AI; task prompt patterns determine how to structure those tasks once selected.
[^b]: [Prompting: Prompt Architecture](01-prompt-architecture.md) — task prompt patterns are the applied form of prompt architecture; the architecture principles manifest as specific patterns per task type.
[^c]: [Tooling: Custom Skills](../Tooling & Configuration/04-custom-skills.md) — high-value task prompt patterns become custom skills; the pattern library is the source from which skills are extracted.