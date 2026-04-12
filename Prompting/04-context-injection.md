## Context Injection: Getting the Right Information to Claude at the Right Level

**Related to:** [Prompting Overview](00-overview.md) — Pattern 3 · [Workflows: Context Engineering](../Workflows/03-context-engineering.md)[^a] · [Tooling: CLAUDE.md Configuration](../Tooling & Configuration/01-claude-md-configuration.md)[^b] · [Documentation: Architecture Decision Records](../Documentation/01-architecture-decision-records.md)[^c]

---

## Overview

Context injection is not about giving Claude more information — it is about giving it the right information at the right level of the session hierarchy. Claude Code operates within a three-layer context architecture: the persistent team layer (CLAUDE.md), the feature layer (spec.md or equivalent), and the per-prompt layer (what you inject in the message itself). Each layer has a different scope and a different retention life. Information placed at the wrong level either fails to apply when it should, or clutters every session with details irrelevant to the work at hand.[^1]

The most consequential distinction in context injection is between what Claude can read itself and what it cannot. Architecture decisions, team conventions, and stable project structure can all be expressed in files that Claude reads on demand. The engineer's job is to ensure those files exist, are accurate, and are referenced correctly — not to re-explain their contents in every prompt. What cannot be read from files is what belongs in the prompt: the current intent, the specific task boundary, the decisions made this session that are not yet captured anywhere.[^2]

---

## Section 1: What Context Injection Is

**Description:** Context injection is the deliberate act of providing Claude with information it cannot obtain from files or prior conversation. It operates at three distinct levels. CLAUDE.md-level context is global: it applies to every session in the project and encodes the team's architectural conventions, style rules, toolchain, and workflow norms. This level is maintained by the architect and changes infrequently. Spec.md-level context is feature-scoped: it captures requirements, constraints, and decisions relevant to a feature branch or sprint-bounded body of work. Session-level context is ephemeral: it is what the engineer injects per prompt or per session to bridge the gap between what is in the files and what Claude needs to know right now.[^3]

Understanding the hierarchy prevents two opposite errors. The first is over-injection: dumping everything into each prompt because it feels safer. The second is under-injection: assuming Claude retains information from a previous session or infers constraints from ambient project structure. Claude has no persistent memory across sessions and cannot read files it has not been shown. Both errors degrade output quality, but in opposite directions — over-injection floods signal, under-injection leaves Claude operating on defaults. The hierarchy exists to make neither necessary: the right information, already encoded at the right level, is available without per-prompt repetition.[^4]

**Recommended Practice:**
- Before writing a prompt, audit which of the three layers is responsible for the context it needs. If the context is a team convention, it belongs in CLAUDE.md. If it is a feature constraint, it belongs in spec.md. If it is specific to this prompt, inject it in the prompt. Injecting CLAUDE.md-level context into every prompt is a sign of a missing or incomplete CLAUDE.md.[^1]
- Treat the three layers as authoritative and non-redundant: do not repeat CLAUDE.md content in spec.md, and do not repeat spec.md content in individual prompts. When content is duplicated across layers, conflicts emerge as the layers drift, and Claude receives contradictory instructions.[^3]
- When onboarding a new project with an existing codebase, the first CLAUDE.md investment is in cataloguing what context currently lives in engineers' heads and is not in any file. That implicit context — naming conventions, deprecated patterns to avoid, the specific shape of the team's testing philosophy — is the highest-priority CLAUDE.md content.[^5]
- Review CLAUDE.md quarterly against what engineers are actually injecting in prompts. If the same context appears in multiple prompts per week, it should be in CLAUDE.md. If content in CLAUDE.md is never referenced by prompts or outputs, it may be outdated or misplaced.[^2]

---

## Section 2: File and Codebase Context Injection

**Description:** The `@` mention is Claude Code's primary mechanism for file-level context injection. Typing `@src/api/handlers/users.ts` in a prompt causes Claude to read and incorporate that file's contents at the point of reference. This is more precise than copying file contents into the prompt — the reference is explicit, the file is read at its current state, and the prompt remains readable to a human engineer reviewing it later. The `@` pattern works for individual files, directories, and URLs, and multiple `@` references can appear in a single prompt.[^6]

The context flood anti-pattern (documented in Prompt Anti-Patterns) applies with particular force to file injection. An endpoint implementation task that requires understanding one handler pattern does not require the full API module injected. Injecting the single file that exemplifies the pattern — `@src/api/handlers/users.ts` — produces better-calibrated output than injecting `@src/api/` and trusting Claude to extract the relevant signal. Specificity of file reference is a proxy for clarity of thought about what context is actually relevant.[^7]

**Recommended Practice:**
- Use `@` references for file context rather than copying file contents into prompts. The reference is explicit, the file is always current, and the prompt stays readable. Reserve direct content pasting for cases where you need to reference a specific excerpt rather than the full file.[^6]
- For architectural context, maintain an `ARCHITECTURE.md` file at the repository root and reference it by `@` when context about the overall system structure is relevant. This keeps architectural documentation as a readable, version-controlled artifact rather than something that must be re-explained per prompt.[^3]
- Avoid injecting files whose relevance to the task you cannot articulate. If the reason for including a file is "Claude might need it," the file probably belongs in CLAUDE.md (if it is always relevant) or should be omitted (if its relevance is speculative). Every injected file adds context noise that competes with the task-specific signal.[^4]
- When the relevant context spans multiple files with shared structure (e.g., three handler files that all exemplify the same pattern), inject the most canonical example and describe the pattern explicitly rather than injecting all three. One precise example with a clear description transfers the pattern more efficiently than three examples without description.[^1]

---

## Section 3: Session-Level Context Setup

**Description:** Session-level context setup is the practice of front-loading a session with the context it will need before issuing any task prompts. The canonical form is the `/init` workflow: starting a session by asking Claude to read CLAUDE.md and any relevant architecture or spec files before proceeding to task work. This ensures the session is calibrated to the project's conventions before the first task prompt rather than discovering mismatches mid-task and correcting them retroactively.[^8]

The reason session-level setup matters is that Claude Code has no persistent memory across sessions. A session that produced excellent output last Tuesday had that quality because of the context it was given. A new session started Wednesday is not inheriting that context — it is starting fresh with only what is in the files and what the engineer provides. Teams that understand this design their context files to make session startup fast and reliable; teams that do not understand it repeat orientation context in every prompt or wonder why output quality varies unpredictably across sessions.[^9]

**Recommended Practice:**
- Begin every substantive session with a context-loading prompt before the first task prompt. A reliable pattern: "Read CLAUDE.md and @ARCHITECTURE.md. Confirm what you understand about our stack, conventions, and current work context before we begin." This surfaces misunderstanding at session start rather than mid-task.[^8]
- Keep session startup fast by keeping CLAUDE.md well-organized and eliminating outdated content. A CLAUDE.md that requires five minutes to read is a sign of accumulated cruft; the goal is a document Claude can absorb in a single context pass that gives it reliable orientation for the project.[^5]
- Do not rely on model behavior to carry context between sessions. Even in long-running development work where you open new sessions daily, treat each session as starting from zero. If context from yesterday's session matters today, encode it in the appropriate persistent file (CLAUDE.md, spec.md, or an ADR) before closing the prior session.[^2]
- For sessions that will involve multiple task types (implementation, then refactoring, then test generation), load the relevant spec.md at the start so all task phases operate within the same feature context. Switching context mid-session is less reliable than establishing it at the start.[^3]

---

## Section 4: Dynamic Context Injection via Hooks

**Description:** Claude Code hooks allow context to be injected automatically before each prompt via the `UserPromptSubmit` event. A `UserPromptSubmit` hook runs before Claude processes the user's message and can prepend information to the prompt — making it possible to inject current git status, the active branch, recent error logs, or sprint task context without the engineer having to provide this manually. Dynamic injection via hooks converts context that engineers currently have to remember to include into context that appears automatically when it is relevant.[^10]

The practical value of dynamic injection is consistency. An engineer who always forgets to mention the current branch when asking about CI failures can have the current branch injected automatically. A team that wants every prompt to include the output of `git status --short` so Claude always knows what files have changed can configure this once in a hook rather than building the habit across every engineer individually. Hooks make the good context injection practice the default practice, not the disciplined practice.[^11]

**Recommended Practice:**
- Implement a `UserPromptSubmit` hook that injects `git status --short` and the current branch name before each prompt for any project where branch and file-change state are frequently relevant. This costs nothing at runtime and ensures Claude always has an accurate view of the workspace state.[^10]
- For projects with active sprint tracking (Jira, Linear, GitHub Projects), implement a hook that appends the current sprint's active task list to session context at startup. Knowing what sprint tasks are in flight helps Claude provide implementation suggestions aligned with current priorities without the engineer having to explain the sprint context.[^12]
- Keep injected context concise. A hook that injects a multi-page error log before every prompt degrades quality via context flooding as surely as manual over-injection. Pre-process injected content to its essential signal: the last ten lines of an error log rather than the full log, the current branch and dirty files rather than the full git diff.[^4]
- Document all active hooks in CLAUDE.md under a "Session Hooks" section so engineers know what context is being injected automatically. Invisible automatic injection is a debugging hazard — when output seems oddly calibrated to context that was not explicitly provided, engineers need to know where it came from.[^10]

---

## Section 5: Context Compression and Prioritization

**Description:** Long sessions accumulate context. Each exchange adds to the context window — the implementations, the corrections, the intermediate outputs, the discarded directions. As a session approaches the context window limit, Claude's attention distributes across an increasingly large body of prior content, and the most recent, most relevant instructions compete with the accumulated history for attention. The `/compact` command addresses this by asking Claude to summarize the session into a compressed representation, discarding the raw exchanges while preserving the key decisions and current state.[^13]

Not all context is equally worth preserving when compressing. The decisions — what patterns were adopted, what constraints were agreed on, what the current implementation state is — are high-value. The path to those decisions — the early attempts, the corrections, the detours — is low-value once the destination is known. Good context compression preserves the former and discards the latter. Engineers who use `/compact` effectively think of it as writing a handoff note to the next phase of the session: what does Claude need to know to continue productively, without the history of how it got here?[^14]

**Recommended Practice:**
- Use `/compact` proactively at natural session phase transitions — after a planning phase completes and before implementation begins, or after one feature is complete and before starting the next. Compressing at a clean boundary produces better-preserved context than compressing mid-task.[^13]
- Before invoking `/compact`, explicitly state what must be preserved: "Compact this session. Preserve: the agreed implementation plan, the authentication approach we settled on, and the three constraints from CLAUDE.md we confirmed apply here. Discard: the early exploration of the alternative approach and the draft code that was rejected." Explicit preservation instructions produce more useful compact summaries than generic compression.[^8]
- When a session is approaching context limits without a clean compression point, consider whether it is better to `/compact` now or to close the session, update the relevant spec.md with the decisions made, and start a fresh session that reads from the updated spec. Fresh-session-with-updated-files is often higher quality than heavily compressed continuation.[^5]
- Add context compression checkpoints to long-task templates (see Section 5 of 05-iterative-refinement.md). Tasks that reliably take long sessions should have a planned compression point as part of the task template, not as an emergency measure when the context limit is imminent.[^14]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Context Hierarchy | Audit which context currently lives in prompts and should be in CLAUDE.md; migrate it | Architect |
| File and Codebase Injection | Establish `@ARCHITECTURE.md` convention; add `@` reference examples to prompt templates | Architect |
| Session-Level Setup | Add `/init` workflow to onboarding docs and command library as the session-start standard | Individual engineers |
| Dynamic Injection via Hooks | Implement `UserPromptSubmit` hook for git status and branch; document in CLAUDE.md | Architect |
| Context Compression | Add compression checkpoints to long-task templates; train engineers on `/compact` usage | Individual engineers |

---

[^1]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    Context hierarchy discipline: the three-layer model for context placement and how misplaced context degrades output quality; the over-injection and under-injection failure modes.

[^2]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    CLAUDE.md scope, content, and maintenance; session memory non-persistence; minimum sufficient context principle applied to file injection.

[^3]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    Three-layer context architecture; non-redundancy discipline across layers; session-start context setup as the primary driver of consistent session quality.

[^4]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    Context flood as the primary failure mode of over-injection; specificity of file reference as a quality signal; keeping injected context proportional to task scope.

[^5]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
    CLAUDE.md onboarding investment priorities; implicit team knowledge as the highest-priority CLAUDE.md content; session startup speed as a CLAUDE.md quality metric.

[^6]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    `@` mention syntax for file, directory, and URL context injection; the difference between file references and content pasting; multiple `@` references in a single prompt.

[^7]: Fannar Steinn Aðalsteinsson et al. — "Rethinking Code Review Workflows with LLM Assistance: An Empirical Study," arXiv:2505.16339, May 22, 2025. https://arxiv.org/abs/2505.16339
    Context specificity as a proxy for task clarity; how over-broad file injection reduces the precision of context-dependent outputs compared to single well-chosen examples.

[^8]: Anthropic — "Hooks Reference," Claude Code Documentation, 2026. https://code.claude.com/docs/en/hooks-reference
    `UserPromptSubmit` hook mechanics; `/compact` compression command; session-start context verification patterns.

[^9]: Sreecharan Sankaranarayanan — "Towards Reliable AI Code Agents: A Framework for Evaluating Context Window Management," arXiv:2602.20206, February 2026. https://arxiv.org/abs/2602.20206
    Session memory non-persistence: empirical evidence that output quality differences across sessions trace primarily to differences in context provided, not model variation; session-start context as the primary quality lever.

[^10]: Kyros — "The Vibe Coding Crisis: How AI-Generated Technical Debt Is Costing Companies Millions," March 2026. https://usekyros.ai/blog/vibe-coding-crisis-ai-technical-debt
    Hook-based automatic context injection as a consistency mechanism; converting good context practice from individual discipline to team default via hooks.

[^11]: Ravikanth Konda — "Patterns for Effective AI-Assisted Software Development," International Journal of AI in Business, Data and Cloud Management Systems, February 2026. https://ijaibdcms.org
    Dynamic context injection patterns: systematic comparison of manual vs. hook-based context injection and the consistency improvement from automation; sprint context as a high-value injection candidate.

[^12]: Judy Shen and Alex Tamkin — "How Instruction Following Affects Context Use in Large Language Models," Anthropic / arXiv:2601.20245, January 2026. https://arxiv.org/abs/2601.20245
    Context attention dynamics: how automatic injection of structured, concise context (branch, task list) improves instruction-following accuracy compared to implicit reliance on model defaults.

[^13]: Anthropic — "Managing Long Sessions," Claude Code Documentation, 2026. https://code.claude.com/docs/en/managing-long-sessions
    `/compact` mechanics and timing; context window management; compressing at clean phase boundaries vs. emergency compression; fresh-session-with-updated-files as an alternative to heavy compression.

[^14]: Dex Horthy (YC Root Access) — "Advanced Context Engineering for Agents," YouTube, August 2025. https://www.youtube.com/watch?v=IS_y40zY-hc
    - Context compression strategy: when to use `/compact` vs. starting a new session with updated files; the handoff-note mental model for producing useful compact summaries
    - Phase-boundary compression: demonstration of compressing at planning-to-implementation transition and the quality difference vs. compressing mid-implementation under pressure
    - Context triage: how to explicitly specify what to preserve in a `/compact` call and why generic compression produces lower-quality summaries than directed compression

[^15]: Sabrina Ramonov — "CLAUDE CODE FULL COURSE," YouTube, February 17, 2025. https://www.youtube.com/watch?v=fYX6hHC9FhQ
    - Session setup workflow: the `/init` pattern in practice — loading CLAUDE.md and architecture files before task work and the calibration difference it produces
    - `@` mention context injection: live demonstration of file reference vs. content paste for codebase context and the readability and precision tradeoffs
    - Dynamic hook injection: configuring `UserPromptSubmit` to prepend git status to every prompt and the session quality consistency improvement it produces


[^a]: [Workflows: Context Engineering](../Workflows/03-context-engineering.md) — context injection is a specific technique within the broader context engineering discipline; the two documents describe the practice at different levels of specificity.
[^b]: [Tooling: CLAUDE.md Configuration](../Tooling & Configuration/01-claude-md-configuration.md) — CLAUDE.md is the primary context injection mechanism for session-persistent information; explicit injection supplements what CLAUDE.md already provides.
[^c]: [Documentation: Architecture Decision Records](../Documentation/01-architecture-decision-records.md) — pre-loading ADR context into sessions is a primary use case for context injection; the two documents describe the artifact and its injection pattern.