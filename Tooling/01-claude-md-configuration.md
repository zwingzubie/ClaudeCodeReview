## CLAUDE.md Configuration: The Team's Persistent Context Layer

**Related to:** [Tooling Overview](00-overview.md) — Tool 1 · [Documentation: Architecture Decision Records](../Documentation/01-architecture-decision-records.md)[^a] · [Issues: Architectural Drift](../Issues/03-architectural-drift.md)[^b] · [Workflows: Context Engineering](../Workflows/03-context-engineering.md)[^c] · [Governance: Review Policies](../Governance/01-review-policies.md)[^d]

---

## Overview

CLAUDE.md is not a configuration file in the traditional sense. It is a standing instruction set that every Claude Code session reads before doing anything — a mechanism for injecting the team's accumulated architectural knowledge, conventions, and corrections into every AI session without requiring individual engineers to re-specify them. The difference between a team that has maintained a CLAUDE.md for six months and one that has not is visible in the consistency of AI-generated output: the first team's sessions align with their conventions; the second team's sessions produce implementations that pattern-match against training data defaults, which are often wrong for the specific codebase.[^1]

This memo goes deeper than the overview's summary. It covers what to put in CLAUDE.md, what not to put in it, how to structure it for maintainability, how to use the import system to keep it from growing unwieldy, and how to test whether it is working. The file is the foundation on which the rest of the tooling infrastructure depends; getting it right has a higher return than any other single tooling investment.

---

## Section 1: What Belongs in CLAUDE.md

**Description:** The purpose of CLAUDE.md is to give Claude knowledge it cannot derive from reading the codebase — the team's decisions, conventions, and constraints that are not represented in any file. Anything Claude would already get right by reading the code does not belong in CLAUDE.md; including it dilutes the file with noise that crowds out the rules that actually matter.[^2]

The most valuable content categories are: stack and dependency constraints (what technologies are in use and which alternatives are prohibited), architectural patterns (the naming conventions, module structure, and design patterns the team has standardized on), active corrections (the specific mistakes Claude made in past sessions, captured as "do not" rules), testing requirements (what must be tested and at what level before any PR is considered complete), and security invariants (patterns that are explicitly forbidden for security reasons regardless of apparent functionality).[^3]

The least useful content categories are: general best practices Claude already knows (prefer async/await over callbacks — Claude does this by default), documentation requirements Claude follows without prompting (add JSDoc comments — Claude already does this), and explanations of why rules exist (Claude does not need to understand your reasoning, only follow the instruction; save the "why" for human documentation).[^2]

**Recommended Practice:**
- Start CLAUDE.md with the highest-signal content: the three to five things that, if Claude does not know them, will produce the most wrong implementations. For most teams, this is the stack constraints, the authentication pattern, and the top one or two corrections from prior sessions.[^3]
- Apply the pruning test monthly: for each instruction, ask whether removing it would cause Claude to make a mistake. If the answer is no, delete it. The file should shrink over time as Claude's baseline behavior improves — not grow indefinitely.[^2]
- Distinguish between "never do X" instructions (hard prohibitions) and "prefer X over Y" instructions (soft guidance). Hard prohibitions belong in CLAUDE.md; soft guidance often produces better results when embedded in specific task prompts rather than as global instructions that apply to every session.[^4]
- Keep a "graveyard" comment at the bottom of CLAUDE.md for instructions that were removed after Claude began following them naturally — this prevents re-adding instructions that were correctly pruned, a common pattern when multiple engineers update the file.[^1]

---

## Section 2: File Structure and Organization

**Description:** An unstructured CLAUDE.md is harder to maintain, harder to prune, and more likely to have contradictions develop between instructions added at different times by different engineers. Structure enables the architect to reason about the file as a whole, not just add to the end. It also makes it possible to identify which category of instruction is consuming most of the file's effective attention — a file where 80% of the content is architectural patterns and only 5% is security invariants may have the wrong balance for a team that has recurring security issues.[^5]

Anthropic's guidance recommends organizing CLAUDE.md with clear sections for different instruction types and using the import system to reference separate files for large instruction categories rather than embedding everything inline. A CLAUDE.md that imports a `git-workflow.md`, a `testing-requirements.md`, and a `security-invariants.md` is shorter, more navigable, and easier to maintain than one where all these categories are interleaved in a single document.[^2]

**Recommended Practice:**
- Structure the main CLAUDE.md with five or six top-level sections using header markup: Stack and Technologies, Architectural Patterns, Testing Requirements, Security Invariants, Active Corrections, and Prohibited Patterns. Keep each section to under ten instructions; if a section grows beyond ten, promote its content to an imported file.[^2]
- Use CLAUDE.md imports (`@path/to/file`) to reference separate, more detailed instruction files. Keep the main file as a quick-reference document and the imported files as the detailed specifications. Engineers reading the main file should be able to understand the team's complete instruction set in three minutes.[^5]
- Date-stamp instructions when they are added: a line like `# Added 2026-01 after authentication PR incident` makes it possible to identify instructions that were added under pressure and may need reconsideration once the incident is resolved.[^3]
- Store CLAUDE.md in the repository root and configure git to require architect review for any PR that modifies it. The file is load-bearing for every engineer's sessions; unauthorized or poorly-considered changes affect the whole team.[^1]

---

## Section 3: Team Ownership and Update Cadence

**Description:** CLAUDE.md's value compounds with deliberate maintenance. A file written at setup and never updated reflects the team's conventions as they were when it was created, not as they are now. As the codebase evolves, the stack changes, and new mistakes surface in AI sessions, a static CLAUDE.md progressively diverges from the team's actual needs — and eventually becomes a source of noise rather than signal.[^6]

Boris Cherny describes the right update cadence as continuous: every time Claude makes a mistake in a session, the correction goes into CLAUDE.md immediately so it does not happen again in any subsequent session by any engineer.[^1] For a team of 11, this requires a clear ownership model — without it, every engineer adds corrections in their own style, the file accumulates contradictions, and the architect who is nominally responsible has no visibility into what changed and why.

A sole-owner model introduces a practical bottleneck: when the architect is unavailable, engineers have no clear path for urgent corrections. This tension requires an explicit resolution in the ownership policy, not just an assumption that PRs will be reviewed quickly.[^14]

**Recommended Practice:**
- Assign the architect sole write access to CLAUDE.md (enforced through git CODEOWNERS). Engineers identify corrections in session logs or PR reviews and propose them to the architect as a two-line PR rather than editing the file directly. This keeps the update velocity high while maintaining coherence.[^1]
- Define a maximum turnaround time for CLAUDE.md correction PRs — 24 hours during active development is a reasonable default. If the architect cannot review within that window, designate a backup reviewer (typically the senior engineer closest to the affected domain) so the correction pipeline does not stall sessions across the team.[^1]
- Establish a tiebreaker policy for disputed instructions: when engineers disagree about whether a correction belongs in CLAUDE.md vs. a task prompt, the deciding question is scope. If the behavior should be consistent across every session by every engineer regardless of task context, it belongs in CLAUDE.md. If it is appropriate only for specific task types or specific files, it belongs in a task prompt or imported domain file, not the main configuration.[^4]
- At each sprint retrospective, include a standing agenda item: "Were there any recurring AI mistakes this sprint that should become CLAUDE.md instructions?" This creates a regular collection mechanism without requiring engineers to remember to file corrections ad hoc.[^6]
- Review the CLAUDE.md in the monthly AI practice review (see Metrics — Retrospective Cadence): evaluate whether recent AI sessions reflect the instructions in the file, whether any instructions appear to be ignored (a sign the file is too long), and whether any recently added instructions have already become unnecessary.[^4]
- When the codebase undergoes a significant architectural change — a framework migration, a database change, a major API redesign — schedule an explicit CLAUDE.md update session with the architect before AI-assisted development begins in the new context. A file that reflects the old architecture will actively mislead sessions working on the new one.[^6]
- Include CLAUDE.md orientation in the engineering onboarding checklist. New engineers should read the file before their first Claude Code session, understand that it governs all sessions across the team, and know how to propose corrections (via PR to the architect) rather than editing it directly. A five-minute walkthrough from the architect during onboarding is more reliable than assuming the file is self-explanatory.[^1]

---

## Section 4: Import Patterns and Modular Configuration

**Description:** The CLAUDE.md import system allows a main file to reference external instruction files using `@path/to/file` syntax, pulling their contents into the effective context without embedding them inline. This enables a modular configuration architecture: a short main file that provides an orientation layer, backed by detailed instruction files for specific domains (git workflow, testing requirements, API conventions, security invariants). The result is a configuration system that is both navigable and detailed without the length constraints that make monolithic CLAUDE.md files dysfunctional.[^2]

Modular configuration also enables team specialization: the backend lead can maintain `api-conventions.md`, the frontend lead can maintain `component-patterns.md`, and the architect maintains the main `CLAUDE.md` and `security-invariants.md`. Each person maintains the section most relevant to their domain without needing to understand the complete file. The architect retains ownership of the main file's structure and the imports that compose them.[^7]

**Recommended Practice:**
- Structure imports by domain: create separate files for `git-workflow.md`, `testing-requirements.md`, `api-conventions.md`, `component-patterns.md`, and `security-invariants.md` once any of these categories grows beyond eight instructions.[^2]
- Place imported files in a `.claude/` directory at the repository root to separate Claude-specific configuration from project source code. Document the import structure in a brief README within that directory.[^5]
- Use conditional imports when relevant: some instruction files apply only to specific project areas. Instruct Claude in the main CLAUDE.md to apply `api-conventions.md` when working in the `src/api/` directory and `component-patterns.md` when working in `src/components/` — avoiding the cognitive load of irrelevant instructions in scoped sessions.[^2]
- Test the complete import chain by starting a Claude session and asking it to summarize the rules it has been given. The response reveals which imports were loaded, which instructions were retained, and whether the modular structure is achieving the intended coverage.[^4]

---

## Section 5: Testing and Validating CLAUDE.md Effectiveness

**Description:** A CLAUDE.md that has not been tested may not be working. Claude may be ignoring long or contradictory files; instructions may be phrased ambiguously enough that Claude interprets them differently than intended; or the file may have grown to a length where the most important instructions are not receiving adequate attention. Without deliberate testing, the file's effectiveness is assumed rather than verified — and the team continues correcting the same mistakes in sessions because the CLAUDE.md instruction intended to prevent them is being silently ignored.[^2]

Validation has two distinct directions. Negative validation confirms that prohibited behaviors are being blocked — Claude refuses to use an off-limits library, declines to skip a required test type, or flags a request that conflicts with a security invariant. Positive validation confirms that required behaviors are being produced — Claude applies the correct naming convention without being asked, uses the established authentication pattern by default, generates the expected test structure. A CLAUDE.md that passes only negative validation may be blocking the wrong behavior while failing to produce the right one; both directions are necessary for confidence.[^3]

Length is the most common silent failure mode. Anthropic's documentation and empirical testing by practitioners suggest that files exceeding roughly 500 tokens (about 50–60 lines of dense instruction) begin to show attention dilution: instructions near the end of the file receive less weight, and the effective instruction set becomes shorter than the written one. The practical implication is that a CLAUDE.md with 80 instructions is often worse than one with 25, because the 80-instruction file crowds out the 20 that actually matter.[^2][^11]

**Recommended Practice:**
- Create a CLAUDE.md test checklist with two columns: prohibited behaviors (things Claude should refuse or flag) and required behaviors (things Claude should produce by default). A minimal checklist entry looks like: `[ ] Prohibited: uses axios — should suggest fetch or built-in http. [ ] Required: new API routes include input validation using the team's schema library.` Run this checklist in a fresh session after any significant update.[^4]
- For positive validation, give Claude an underspecified implementation task in a fresh session — no task prompt beyond the feature description — and evaluate whether the output reflects team conventions without prompting. If Claude produces the right patterns without being asked, the CLAUDE.md is working. If it produces training data defaults, the relevant instructions need revision or repositioning.[^3]
- If an instruction fails validation (Claude does not follow it), diagnose before rewriting. Common causes: the instruction is buried too far into a long file, it conflicts with another instruction, or it is phrased as a preference rather than a prohibition. Move the instruction earlier in the file, resolve the conflict, or restate it as a hard prohibition. Address the root cause rather than restating the instruction more forcefully in the same position.[^2]
- Apply the length budget actively: keep the main CLAUDE.md under 500 tokens. Use the token count displayed in a fresh session (ask Claude to report the CLAUDE.md token count at session start) as the enforcement mechanism. When the file exceeds the budget, the next pruning session is not optional — it is load-bearing for the effectiveness of every subsequent session.[^11]
- Use the verbose mode (Ctrl+O during a session) to inspect Claude's internal reasoning when it executes a task. Visible reasoning often reveals whether specific CLAUDE.md instructions were consulted during the task — a direct window into whether the configuration is actively shaping behavior.[^8]
- After any significant rework PR (a PR that required multiple correction cycles to pass review), review the CLAUDE.md to determine whether a new instruction would have prevented the issue. If yes, add it. If no, diagnose whether the root cause was a prompting problem (scope for Prompting sub-topics) or a task type problem (scope for Governance — Sprint Planning).[^6]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Content Strategy | Start with 3–5 high-signal instructions; add pruning test to monthly review | Architect |
| File Structure | Add five section headers; move imports to `.claude/` directory | Architect |
| Team Ownership | Configure git CODEOWNERS; add CLAUDE.md item to sprint retrospective | Architect |
| Import Patterns | Break instructions above 8 per domain into imported files | Architect |
| Testing and Validation | Create CLAUDE.md test checklist; run after every significant update | Architect |

---

[^1]: HowBorisUsesClaudeCode.com — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    Fan-curated resource documenting Boris Cherny's (Claude Code's creator at Anthropic) workflow tips. Continuous update discipline: every session mistake is immediately added as a CLAUDE.md correction ("Compounding Engineering"); the compound effect of six months of maintained corrections vs. a static file.

[^2]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    CLAUDE.md length discipline: the pruning test; what to include vs. what Claude already does correctly; the import system and its use for modular configuration.

[^3]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    CLAUDE.md file structure guidance: section organization, import syntax, scoped instruction patterns for directory-specific behavior.

[^4]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," January 2026. https://addyosmani.com/blog/ai-coding-workflow/
    CLAUDE.md as a behavioral specification document: Osmani describes maintaining a CLAUDE.md with process rules and preferences that shape Claude's output; the distinction between hard prohibitions and soft guidance in instruction design.

[^5]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    Modular context architecture: how team specialization in maintaining different context files enables depth without requiring any single person to own the complete instruction set.

[^6]: Artur Less — "Spec-Driven Development with Claude Code," Level Up Coding / Medium, March 2026. https://levelup.gitconnected.com/spec-driven-development-with-claude-code-1b08184965e3
    CLAUDE.md update cadence during architectural transitions: the risk of a file that reflects a previous architecture actively misleading sessions working in a new context.

[^7]: Vocal/Futurism — "8 AI Code Generation Mistakes Devs Must Fix to Win 2026." https://vocal.media/futurism/8-ai-code-generation-mistakes-devs-must-fix-to-win-2026
    The Constraint Persona Problem: AI defaults to training data patterns when not given explicit architectural constraints; CLAUDE.md as the mechanism for providing those constraints consistently across all team sessions.

[^8]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    Verbose mode (Ctrl+O): how to inspect Claude's reasoning during sessions to verify that CLAUDE.md instructions are being actively consulted vs. ignored in the session's internal reasoning chain.

[^9]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
    CLAUDE.md in CI contexts: how the same configuration file governs both interactive developer sessions and automated pipeline sessions, ensuring consistent behavior across all Claude-mediated operations.

[^10]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
    Plan-first workflow interaction with CLAUDE.md: how the configuration file shapes the Plan phase of the Explore-Plan-Implement loop by pre-constraining the space of architecturally acceptable plans before implementation begins.

[^11]: Dex Horthy (YC Root Access) — "Advanced Context Engineering for Agents," YouTube, August 2025. https://www.youtube.com/watch?v=IS_y40zY-hc
    - Modular CLAUDE.md design: using import directives to build a maintainable, domain-specific instruction architecture that scales with team size and codebase complexity
    - Length management: empirical demonstration of how context window attention distributes across instructions of varying depth and placement in the file
    - Testing configuration effectiveness: techniques for verifying that instructions are being followed rather than silently ignored in long or complex CLAUDE.md files

[^12]: Sabrina Ramonov — "The ULTIMATE Claude Code Tutorial," YouTube, February 17, 2026. https://www.youtube.com/watch?v=fYX6hHC9FhQ
    - Step 6 (CLAUDE.md): complete walkthrough of file structure, section organization, import patterns, and the five-category architecture for a maintainable team configuration file
    - Team ownership model: how to configure CODEOWNERS, the PR review process for CLAUDE.md changes, and the standing retrospective agenda item for collecting corrections
    - Validation: the test checklist approach for verifying CLAUDE.md effectiveness after updates

[^14]: Addy Osmani — "Your AI coding agents need a manager," January 2026. https://addyosmani.com/blog/coding-agents-manager/
    Team-level coordination challenges with AI coding agents and the need for explicit governance structures rather than informal assumptions about review velocity and availability. See also: TrueFoundry — "Claude Code Governance: Building an Enterprise Usage Policy from Scratch," March 2026. https://www.truefoundry.com/blog/claude-code-governance-building-an-enterprise-usage-policy-from-scratch — addresses ownership models and escalation paths in Claude Code deployments.

[^a]: [Documentation: Architecture Decision Records](../Documentation/01-architecture-decision-records.md) — ADR AI Constraints fields are the source of truth for CLAUDE.md entries; ADR acceptance triggers CLAUDE.md updates as a required workflow step.

[^b]: [Issues: Architectural Drift](../Issues/03-architectural-drift.md) — CLAUDE.md is the session-time enforcement layer for the architectural constraints that prevent drift; the issue and its countermeasure are paired.

[^c]: [Workflows: Context Engineering](../Workflows/03-context-engineering.md) — CLAUDE.md is the foundational layer of context engineering; the two documents describe the artifact and the discipline around it.

[^d]: [Governance: Review Policies](../Governance/01-review-policies.md) — review policies require that AI-primary PRs reference the CLAUDE.md constraints that governed the session; the two documents are operationally linked at review time.
