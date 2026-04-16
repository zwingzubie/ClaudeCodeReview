## Overview

For our team of 11 — 4 backend engineers, 3 frontend engineers, 1 architect, 1 QA engineer, and 2 product managers — the quality of Claude Code output is not determined solely by the model's capability. It is determined by the quality of the context and instructions engineers provide to it. The same model that produces architecturally consistent, well-tested code in one session produces bloated, convention-violating code in another — and the primary variable is the engineer's prompting practice. A 2026 analysis found that structured prompting approaches produced outputs requiring 60% fewer correction cycles than unstructured requests to the same model.[^1]

This gap between structured and unstructured prompting is not a knowledge problem — most engineers understand that better prompts produce better results. It is a practice problem: in the absence of team norms, engineers default to whatever approach feels natural in the moment, producing the fragmented output described in Issue 7 of the Issues section. Standardizing the team's prompting practice — through shared patterns, a maintained prompt library, and explicit anti-patterns — converts individual prompt-engineering skill into a team asset that every session benefits from equally.[^2]

Six practices are documented below. They build from the architectural (how prompts are structured) through the practical (patterns by task type, context injection) to the preventive (anti-patterns, refinement processes) and organizational (library management and sharing).

---

## Pattern 1: Prompt Architecture

**Description:** A well-structured prompt has identifiable components: a role or persona (what Claude is acting as), a task (what it should produce), constraints (what it cannot do), context (what it needs to know that it cannot read from files), and verification criteria (how it knows the output is correct). Prompts that omit components produce outputs that fill the gaps with defaults — often the wrong defaults for the team's context. The most common omission is verification criteria: prompts that do not specify how Claude should validate its work produce outputs that look plausible but may not function correctly.[^3]

The role and constraint components are where architectural context lives. A prompt that says "implement a new API endpoint" gives Claude latitude to choose patterns from its training data rather than from the team's conventions. A prompt that says "implement a new API endpoint following our route handler pattern, using the shared middleware stack defined in `src/middleware/`" constrains Claude to the team's actual architecture. This distinction is the difference between code that passes tests and code that fits the codebase.[^2]

**Proposed Solution:**
- Adopt a consistent prompt template structure for all non-trivial requests: role/persona → task → constraints → context → verification criteria. Encode this template in the team's foundational custom command and reference it in CLAUDE.md.[^3]
- Always include an explicit verification step in prompts involving implementation: "After implementing, run the test suite and fix any failures before reporting completion." Without this, Claude reports completion when code is written, not when it is verified.[^3]
- For architectural prompts, include explicit pointers to relevant existing files: "Before implementing, read `src/api/users.ts` to understand the handler pattern we use." Prompts that assume Claude knows the codebase without reading it produce outputs based on training data patterns rather than actual code.[^2]
- Standardize constraint language across the team: maintain a short list of architectural no-nos in CLAUDE.md that every prompt inherits implicitly — so engineers don't need to repeat them and new engineers benefit from them automatically.[^4]

---

## Pattern 2: Task-Specific Prompt Patterns

**Description:** Different task types require structurally different prompts. A feature scaffolding prompt needs to establish architectural context before any generation begins; a refactoring prompt needs to specify what should change and what should stay identical; a test generation prompt needs access to the function under test and its dependencies; a security review prompt needs to specify which vulnerability classes to focus on. Using the same generic prompt structure for all task types reduces output quality across all of them.[^5]

Phillip Carter at Honeycomb documented this distinction as the "task suitability framework" — the recognition that AI tools perform materially better on some task types than others, and that structuring prompts to align with those strengths multiplies the quality of results. Tasks that are well-represented in training data (REST endpoints, CRUD operations, unit tests for pure functions) need less constraint in prompts; novel tasks (custom business logic, organization-specific patterns) need explicit context that Claude cannot infer.[^5]

**Proposed Solution:**
- Maintain separate prompt templates for the five core task types: feature scaffolding, refactoring, test generation, security review, and documentation. Store these as named commands in `.claude/commands/` rather than requiring engineers to construct them from memory.[^3]
- For feature scaffolding prompts, begin with an Explore phase before any generation: "Read `[relevant files]` to understand the patterns used, then generate a scaffolding plan for `[feature]`." Do not skip the Explore step under time pressure — it is the step that aligns AI output with actual codebase architecture.[^3]
- For security review prompts, specify the vulnerability classes explicitly: "Review for injection vulnerabilities, hardcoded credentials, authentication bypass paths, and unauthorized data access. Reference our threat model at `docs/security-model.md`." Generic "review for security issues" prompts produce generic, low-precision results.[^7]
- For refactoring prompts, specify both what should change and what should not: "Refactor `[function]` to eliminate the nested conditionals. Do not change the function signature or the return type. Run the existing tests to verify behavior is unchanged." Omitting the preservation constraint often causes Claude to also change interfaces it was not asked to change.[^2]

---

## Pattern 3: Context Injection Strategies

**Description:** Claude Code sessions start with no memory of prior sessions. The quality of context provided at the start of each session determines the ceiling of output quality for that session — not Claude's capability. On a team where relevant context exists in multiple forms (CLAUDE.md, open tickets, architecture docs, prior PR descriptions), the challenge is not having context but knowing which context to inject and how.[^8]

Context injection is most effective when it is structured and scoped. Injecting an entire architecture document into a session that needs one endpoint adds noise that degrades output quality — Claude attends to all provided context roughly equally, and irrelevant context crowds out relevant context. The principle is minimum sufficient context: the smallest set of context that covers everything Claude needs to know that it cannot read from files.[^4]

**Proposed Solution:**
- Define three levels of context injection in the team's prompting practice: global (CLAUDE.md, always present), feature-level (spec.md for the current feature, present for implementation sessions), and task-level (specific files, function signatures, or schema excerpts for narrow sessions). Each level supplements rather than replaces the others.[^4]
- Use the `@` file reference syntax to inject context from specific files rather than copying it into prompts: `@src/api/routes.ts` loads the file contents into context at the point of reference. This keeps prompts readable and ensures the injected context is always current.[^3]
- When the relevant context exists in a PR description, ticket, or architecture note that is not yet in the codebase, use a brief summary preamble: "Background: we are migrating from synchronous to async processing. The following change is one step in that migration. [task description]." This one-sentence framing often eliminates an entire category of architecturally wrong outputs.[^8]
- For sessions requiring organizational context Claude cannot have (product tradeoffs, customer constraints, technical decisions not yet in code), use the UserPromptSubmit hook to inject standard context automatically — removing the dependency on each engineer to remember to provide it.[^9]

---

## Pattern 4: Iterative Prompt Refinement

**Description:** The first prompt in a new task type is rarely optimal. Effective teams treat prompts as code: they evolve through use, are improved when they produce poor output, and are documented with the reasoning behind structural choices. A prompt that produced poor output in one session carries information about how to improve it — that signal is wasted if the prompt is discarded rather than refined and re-used.[^1]

The CLAUDE.md update discipline that Boris Cherny describes — adding a correction every time Claude makes a mistake — is a form of iterative refinement at the session-context level. The same discipline applied to the prompt library creates a compounding asset: each cycle of use-evaluate-refine makes the team's prompts more effective, and that improvement benefits every subsequent user of the command.[^10]

**Proposed Solution:**
- After any session that required more than two correction cycles, identify whether a better initial prompt would have prevented the corrections. If yes, update the relevant team command with the improvement and document why in the command file.[^10]
- Hold a brief "prompt retrospective" as part of the quarterly AI practice review: which commands produced consistently good output, which required frequent manual correction, and what changes would address the failure modes.[^11]
- Treat prompt improvement as a codebase contribution: improvements to team commands should go through PR review with a description of what the old prompt produced, why it was insufficient, and what the revised prompt addresses.[^3]
- Encourage engineers to share effective ad-hoc prompts that haven't yet been codified as commands — the retrospective is the venue for converting these into official team commands rather than leaving them in individual engineers' memories.[^11]

---

## Pattern 5: Prompt Anti-Patterns

**Description:** Certain prompt structures reliably degrade output quality regardless of the task. The most common are: vague task scope ("make this better"), missing constraints (no mention of existing patterns to follow), excessive context (injecting entire documentation sets for a narrow task), missing verification ("implement X" without a test or check to run), and closed-loop prompts (asking Claude to review its own implementation without a fresh context session).[^12]

The "vague improvement" anti-pattern is particularly prevalent under velocity pressure. Engineers short on time often pass code to Claude with minimal guidance, hoping for improvement. The result is code that is changed but not necessarily improved in the ways the engineer intended — and potentially changed in ways that break existing behavior. The time saved not crafting the prompt is spent debugging unexpected changes post-implementation.[^13]

**Proposed Solution:**
- Codify the team's top five prompt anti-patterns in CLAUDE.md as explicit "do not ask Claude to" instructions. This catches the most common failure modes before they produce a session worth debugging.[^12]
- Establish the team norm that any prompt for a non-trivial change must include what should stay the same, not just what should change. This single discipline eliminates most "Claude changed things I didn't ask it to change" incidents.[^2]
- For prompts that have repeatedly produced poor output despite refinement, diagnose whether the task type is suited to AI-first execution. Some task types — novel design decisions, ambiguous requirements, security-critical paths — benefit from human-first drafting with AI assisting in refinement rather than leading.[^5]
- Include anti-pattern examples in onboarding for new team members. The most effective way to communicate what a prompt should not look like is to show the specific failure mode it produces.[^13]

---

## Pattern 6: Prompt Library Management

**Description:** A team prompt library is not a collection of templates — it is a versioned, maintained artifact that encodes the team's accumulated knowledge about how to work effectively with Claude Code on their specific codebase. The distinction matters: templates can be written once and forgotten; a library requires active maintenance, review, and deprecation as the codebase and team conventions evolve.[^3]

Teams without managed prompt libraries develop prompt fragmentation: eight engineers using eight different approaches for the same task type, with no shared baseline and no mechanism for collective improvement. The Pragmatic Engineer's 2026 AI tooling survey found that this fragmentation is particularly pronounced at companies with under 20 engineers — exactly our team's size — where prompt standardization is rarely treated as an engineering priority until the costs of inconsistency become visible.

**Proposed Solution:**
- Version-control the prompt library in `.claude/commands/` alongside CLAUDE.md. Include a changelog in the commands directory README explaining what changed and why for each update.[^3]
- Define clear ownership: the architect maintains the CLAUDE.md and core command library; individual engineers maintain task-type commands in their area (backend engineers own API commands, frontend engineers own component commands). All changes go through architect review.[^10]
- Establish a command lifecycle: proposed → reviewed → active → deprecated. Commands that produce poor output consistently should be deprecated and replaced rather than silently overridden with individual workarounds.[^3]
- Make the prompt library discoverable: at onboarding, new engineers receive a walkthrough of available commands and an explanation of why each is structured as it is. This makes the library usable from day one and creates the feedback mechanism for new engineers to identify gaps.[^11]

---

## Summary of Recommended Actions

| Prompting Practice | Immediate Action | Owner |
|---|---|---|
| Prompt Architecture | Document standard five-component template; encode in CLAUDE.md | Architect |
| Task-Specific Patterns | Build five-command library (scaffold, refactor, test, security, docs) | Engineering team |
| Context Injection | Define three-tier context model; configure UserPromptSubmit hook | Architect |
| Iterative Refinement | Add prompt retrospective to quarterly AI practice review | Engineering team |
| Prompt Anti-Patterns | Codify top five anti-patterns in CLAUDE.md | Architect |
| Library Management | Establish command lifecycle and ownership model | Architect |

---

[^1]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
 Structured prompting discipline: the argument that systematic upfront specification reduces correction cycles more than any other individual practice; team prompt libraries as shared infrastructure.

[^2]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
 Prompt structure guidance, constraint specification for refactoring tasks, verification criteria requirements, and the CLAUDE.md-as-global-constraint model.

[^3]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
 Explore-before-implement prompting discipline, `@` file reference syntax, verification step requirements in implementation prompts, and feature-level spec.md context injection.

[^4]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
 Minimum-sufficient-context discipline; CLAUDE.md as global constraint layer that prompts inherit implicitly; context scoping to avoid diluting relevant instructions with noise.

[^5]: Phillip Carter — "How I Code With LLMs These Days," Honeycomb, March 2025. https://www.honeycomb.io/blog/how-i-code-with-llms-these-days
 Task suitability framework: which task types benefit from AI-first prompting vs. human-first drafting; structured prompts aligned to AI strengths vs. those compensating for AI limitations.

[^7]: Veracode — "Spring 2026 GenAI Code Security Update: Despite Claims, AI Models Are Still Failing Security," March 24, 2026. https://www.veracode.com/blog/spring-2026-genai-code-security/
 Security prompt specificity: why generic "review for security" prompts produce low-precision results; vulnerability class enumeration and threat model reference as prompt structure requirements.

[^8]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
 Context engineering as the primary discipline: "The real skill in working with coding agents is no longer prompt design — it's context engineering." Minimum sufficient context vs. injecting everything available.

[^9]: Anthropic — "Hooks Reference," Claude Code Documentation, 2026. https://code.claude.com/docs/en/hooks-reference
 UserPromptSubmit hook for automatic context injection; removing the dependency on engineers to supply standard context manually at each session start.

[^10]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
 Iterative prompt improvement as a practice: updating CLAUDE.md and team commands as a response to AI mistakes; the compound value of each cycle of refinement.

[^11]: Dex Horthy (YC Root Access) — "Advanced Context Engineering for Agents," YouTube, August 2025. https://www.youtube.com/watch?v=IS_y40zY-hc
 - Shared context as team infrastructure: treating CLAUDE.md and command libraries as architectural artifacts that compound value over time
 - Prompt retrospectives: structured review of what context approaches worked, what produced poor output, and how to encode improvements into shared configuration
 - Context drift: how prompt quality degrades without active maintenance as the codebase evolves

[^12]: DEV Community — "AI Is Creating a New Kind of Tech Debt — And Nobody Is Talking About It," March 2026. https://dev.to/harsh2644/ai-is-creating-a-new-kind-of-tech-debt-and-nobody-is-talking-about-it-3pm6
 Prompt anti-patterns and their contribution to technical debt: vague improvement prompts, missing constraints, and closed-loop review all identified as primary sources of AI-generated technical debt.

[^13]: daily.dev — "Vibe Coding in 2026: How AI Is Changing the Way Developers Write Code," April 2026. https://daily.dev/blog/vibe-coding-how-ai-changing-developers-code
 The velocity trap: why vague prompts feel faster in the moment but produce rework that costs more time than a structured prompt would have required.

[^15]: Sabrina Ramonov — "CLAUDE CODE FULL COURSE," YouTube, February 17, 2025. https://www.youtube.com/watch?v=fYX6hHC9FhQ
 - Step 1 (Custom Skills): structuring slash commands with role, task, constraints, and verification components for consistent output across team members
 - Step 2 (Prompt Library): building and maintaining a team command library in `.claude/commands/`; the lifecycle from draft to reviewed to active
 - Step 5 (Anti-Patterns): the specific prompt constructions that reliably degrade output quality and how to replace them with structured alternatives

