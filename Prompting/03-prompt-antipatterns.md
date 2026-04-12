## Prompt Anti-Patterns: What Degrades Output and Why

**Related to:** [Prompting Overview](00-overview.md) — Pattern 5 · [Issues: Prompt Fragmentation](../Issues/07-prompt-fragmentation.md)[^a] · [Prompting: Prompt Architecture](01-prompt-architecture.md)[^b] · [Governance: Review Policies](../Governance/01-review-policies.md)[^c]

---

## Overview

Anti-patterns are not just bad practices — they are practices that produce reliably worse outcomes than their alternatives, often while feeling correct in the moment. Prompt anti-patterns are especially persistent because the feedback loop is long: a vague prompt feels faster to write, and the inferior output it produces shows up ten minutes later in review or the next day in a bug report — far enough from the prompt that the connection between cause and effect is not obvious. Teams that do not name and study their anti-patterns repeat them indefinitely.[^1]

This memo identifies six prompt anti-patterns that consistently degrade output quality in AI-assisted development: the vague improvement request, the missing preservation constraint, the closed-loop review, the context flood, the permission skip, and the single-shot agentic delegation. For each, it explains the mechanism by which it degrades quality, the failure mode it produces, and the structural alternative that addresses the root cause. These anti-patterns inform the team's prompt quality review process and are candidates for CLAUDE.md prohibitions.

---

## Section 1: The Vague Improvement Request

**Description:** "Make this better." "Clean this up." "Improve the code quality in this module." These requests are common under time pressure because they are fast to write and feel productive. They are anti-patterns because they delegate the definition of "better" entirely to Claude, which fills the gap with training data norms that may not match the team's context. The result is code that is different — sometimes genuinely improved, often just different — with no way to evaluate whether the changes actually achieved what the engineer wanted.[^2]

The failure mode is insidious: the code may look better in isolation while violating team conventions, changing behavior subtly, or introducing patterns that conflict with adjacent modules. The engineer reviews the diff, sees no obvious problems, and merges it. Three sprints later, someone modifying the "improved" module discovers it uses a pattern inconsistent with everything around it — and traces the change back to a vague improvement request months earlier.[^3]

**Recommended Practice:**
- Replace vague improvement requests with specific, evaluable objectives: "Extract the error handling logic from `processOrder` into a shared `handlePaymentError` function without changing the function signature or test behavior." The specificity is not extra overhead — it is what makes the output useful rather than merely different.[^2]
- When the improvement goal is genuinely underspecified (e.g., early in a refactoring discussion), use a planning prompt rather than an improvement request: "Review this module and identify three specific improvements with tradeoffs. Do not make changes yet." This converts a vague request into a structured planning conversation before any code changes.[^4]
- If an engineer realizes mid-session that they have been issuing vague improvement requests, `/clear` and restart with a specified objective rather than continuing to refine vague output. Each correction round of vague output adds context that further obscures what was actually wanted.[^1]
- Codify the anti-pattern in CLAUDE.md: "If asked to improve code without specific objectives, ask for clarification before making changes." This prompts Claude to surface the missing specification rather than filling it with defaults.[^5]

---

## Section 2: The Missing Preservation Constraint

**Description:** The missing preservation constraint is the most common structural failure in refactoring and enhancement prompts. It occurs when a prompt specifies what should change without specifying what must stay the same. Claude, given latitude to improve as well as implement, will often make "improvements" to adjacent code that was not the subject of the request — renaming methods it finds inconsistently named, simplifying logic it finds complex, refactoring error handling it finds inelegant. Each individual change may be reasonable in isolation; collectively, they produce a diff far larger than intended and with no clear scope boundary.[^6]

The technical debt cost of missing preservation constraints is high. A single session without preservation constraints can produce changes across dozens of files. Each change is an implicit decision that was not reviewed. Each decision is a potential source of divergence from the team's conventions. The engineer who submits this as a PR either has to review and explain changes they did not request, or hopes the reviewer does not ask about them.[^7]

**Recommended Practice:**
- Lead every refactoring prompt with preservation constraints before any change directives. The structural pattern: "Without changing [list of what must stay the same], [description of what should change]." The constraint comes first to establish the boundary before describing the change.[^6]
- Enumerate preservation constraints at multiple levels: API surface (function signatures, return types), behavior (existing test outcomes), and scope (which files may and may not be modified). Different levels of constraint catch different classes of unintended change.[^2]
- After any refactoring session, review the diff specifically for changes outside the intended scope before committing. If Claude made changes you did not request, investigate whether they are correct before accepting them — they may be improvements, or they may be violations of constraints that were not specified.[^7]
- For engineers who find themselves frequently writing "why did Claude change X?" in PR comments, diagnose whether their prompts consistently lack preservation constraints. This is the most common root cause of that review pattern.[^3]

---

## Section 3: The Closed-Loop Review

**Description:** A closed-loop review occurs when the engineer asks Claude to review code that the same session (or a session with the same context) generated. The same session that implemented the code has already rationalized its architectural choices; asking it to review the output is asking it to find problems it already implicitly resolved during implementation. The review is not independent, and it does not produce the fresh-perspective findings that make code review valuable.[^8]

The closed-loop anti-pattern is subtle because it feels rigorous. "Before finishing, review this implementation for issues" seems like a quality step. But the issues most likely to be in AI-generated code — architectural misfit, implicit assumptions, edge cases that the implementation silently skips — are precisely the issues that the implementing context has already addressed (correctly or not). The review confirms the implementing session's worldview rather than challenging it.[^9]

**Recommended Practice:**
- For any non-trivial code review, use a separate Claude session with no knowledge of the implementation session. Open a new terminal, start a fresh session with CLAUDE.md loaded, and provide the code to be reviewed without explaining how it was generated. This is the writer/reviewer pattern (Workflow 6) applied at the prompt level.[^9]
- When asking a fresh-context session to review code, specify what it does not know: "Review this implementation. It was generated by an AI session; you have no knowledge of the implementation choices made." This prevents the reviewer session from trying to reconstruct the implementing session's reasoning.[^8]
- If a fresh-context reviewer session is not feasible, use a weaker but better-than-nothing alternative: provide the original specification (the requirements) alongside the implementation and ask "does this implementation fully satisfy the specification?" rather than "review this implementation." Evaluating against a specification is more independent than general review.[^2]
- Train engineers to recognize the closed-loop anti-pattern in PR review: a reviewer who has seen the implementation PR is not providing independent review of the architectural choices. Architectural review is most valuable from reviewers who evaluate the implementation against requirements rather than against the implementation itself.[^1]

---

## Section 4: The Context Flood

**Description:** The context flood occurs when engineers inject far more context into a prompt than the task requires — a full architecture document for a task that involves one endpoint, an entire codebase history for a task that touches two files, a comprehensive system overview for a ten-line bug fix. The intent is to give Claude everything it might need. The effect is to dilute the signal: with too much context, the specific details relevant to the task receive proportionally less attention, and important constraints buried in a long context block are attended to less reliably.[^10]

The failure mode is output that is technically aware of all the injected context but calibrated to none of it with precision. Claude acknowledges the architectural constraints mentioned in the overview but generates code that does not actually apply them. The engineer who injected the context assumes it was absorbed; Claude generated code that pattern-matched on it superficially without internalizing the specific implications for the task at hand.[^11]

**Recommended Practice:**
- Apply minimum sufficient context discipline: provide only the context that is specifically relevant to the task. For an endpoint implementation, that is the relevant route handler pattern and the authentication middleware interface — not the complete API documentation.[^10]
- Use `@` file references for context that Claude needs but should read itself, rather than copying and pasting file contents into the prompt. `@src/api/users.ts` injects the file at the point of reference; the prompt remains readable while the context is precise.[^2]
- When unsure which context to include, ask before flooding: "Before I provide context, which files do you need to see to implement [task]?" This uses Claude's own assessment to scope the context rather than providing everything by default.[^5]
- If a session produced output that did not apply a specific constraint despite the constraint being present in the context, diagnose the root cause: was the constraint buried in a large context block? Rephrasing it as a leading constraint in the prompt (rather than an item in an injected document) will produce more reliable adherence.[^11]

---

## Section 5: The Permission Skip

**Description:** The permission skip occurs when engineers approve Claude permission prompts reflexively — clicking through confirmation dialogs without reading them — because they are focused on the task and the prompts feel like friction. Over time, this makes the permission system invisible: engineers no longer see it as providing information, only as providing delay. A session that receives reflexive approval for every permission prompt can modify files, run commands, and execute operations outside the engineer's intended scope, and the engineer will not notice until the consequences appear in git diff or in production.[^12]

This anti-pattern is documented in behavioral research: a January 2026 Sonar survey found that 52% of developers who state they do not trust AI-generated code accept it without verification. The same bias operates on permission prompts: a stated awareness that Claude might do something unintended does not translate to actual attention to the permission being requested.[^13]

**Recommended Practice:**
- Establish a personal discipline of reading every permission prompt before approving it: what tool is being called, with what parameters, on what target. This is a five-second investment that prevents the class of incidents where Claude modifies something outside the session's intended scope.[^12]
- Use the PreToolUse hook to log all permission prompts to a session file, enabling post-session review of what was approved even if individual approvals were reflexive. This creates the audit trail that reflexive approval would otherwise eliminate.[^14]
- For sessions working near sensitive code (authentication, payment processing, database migrations), configure more restrictive permission defaults that require explicit review rather than defaulting to allow. The added friction in sensitive contexts is proportional to the risk of unreviewed operations there.[^5]
- Brief new engineers on the permission system as part of the Claude Code onboarding: explain what it is providing (visibility into what Claude is about to do), why it matters (Claude can modify files and run commands you did not intend), and what the cost of reflexive approval is (losing the primary oversight mechanism for session operations).[^1]

---

## Section 6: Premature Agentic Delegation

**Description:** Premature agentic delegation occurs when engineers delegate tasks in auto mode (no human interaction) that require human judgment at decision points within the task. A task that seems well-specified may encounter an ambiguous situation mid-execution — an unexpected file conflict, an unclear requirement edge case, an architectural decision that was not anticipated. In a supervised session, Claude surfaces this for human input. In an unsupervised agentic session, Claude resolves it with a default — which may be incorrect — and continues. The engineer returns to find a completed task that also made several incorrect decisions silently.[^15]

The METR productivity studies found that tasks where engineers expected 19% speedup from AI assistance sometimes produced slowdowns, primarily because agentic tasks went wrong in ways that required more debugging time to unwind than the automation time savings.[^16] The root cause was consistently the same: tasks delegated to autonomous execution that encountered decision points requiring human judgment.

**Recommended Practice:**
- Apply the delegability checklist before any agentic session: fully specified requirements (yes/no), narrow file scope (yes/no), high-precedent patterns in codebase (yes/no), existing verification criteria (yes/no), human review step before production effect (yes/no). All five should be yes before delegating autonomously.[^15]
- Prefer supervised sessions over autonomous ones until the team has accumulated evidence that specific task types are reliably handled without human intervention. The first three autonomous executions of a task type should be monitored; subsequent ones can be delegated if the first three required no human correction.[^4]
- Configure agentic sessions to pause and surface ambiguity rather than resolving it silently: add to the prompt "If you encounter a decision point that was not specified in the task, stop and ask before continuing." This converts a fully autonomous session into a supervised-by-exception session that requires attention only when something unexpected occurs.[^15]
- When a previously reliable agentic task type begins producing unexpected outcomes, revert to supervised sessions for that task type before investigating root cause. The investigation and fix can proceed in a supervised session where unexpected decisions are surfaced; continuing to delegate while investigating risks additional silent errors.[^2]

---

## Summary of Recommended Practices

| Anti-Pattern | Root Cause | Structural Fix |
|---|---|---|
| Vague Improvement Request | Missing specific objective | Replace with evaluable objective or planning prompt |
| Missing Preservation Constraint | Incomplete constraint specification | Lead refactoring prompts with "do not change" before "change" |
| Closed-Loop Review | Non-independent review context | Use fresh-context reviewer session for all non-trivial reviews |
| Context Flood | Over-injection of irrelevant context | Apply minimum sufficient context; use `@` references |
| Permission Skip | Reflexive approval behavior | Read every prompt; add PreToolUse audit log hook |
| Premature Agentic Delegation | Delegating tasks with implicit decision points | Apply delegability checklist before autonomous sessions |

---

[^1]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    Anti-pattern persistence: how the long feedback loop between prompt and downstream consequence prevents engineers from connecting poor prompting with the quality problems it produces.

[^2]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    Specific objective patterns, preservation constraints, minimum sufficient context discipline, and the CLAUDE.md clarification request instruction as structural alternatives to each anti-pattern.

[^3]: DEV Community — "AI Is Creating a New Kind of Tech Debt — And Nobody Is Talking About It," March 2026. https://dev.to/harsh2644/ai-is-creating-a-new-kind-of-tech-debt-and-nobody-is-talking-about-it-3pm6
    Vague improvement requests as a primary source of AI-generated technical debt: how "make it better" sessions produce inconsistent changes that accumulate as the kind of technical debt hardest to trace and unwind.

[^4]: Phillip Carter — "How I Code With LLMs These Days," Honeycomb, March 2025. https://www.honeycomb.io/blog/how-i-code-with-llms-these-days
    Supervised vs. autonomous session tradeoffs: the evidence for starting with supervised sessions before delegating autonomously; the planning prompt as an alternative to both vague requests and agentic delegation.

[^5]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    CLAUDE.md clarification request instruction; minimum sufficient context as a session quality practice; sensitive-context permission tightening.

[^6]: Kyros — "The Vibe Coding Crisis: How AI-Generated Technical Debt Is Costing Companies Millions," March 2026. https://usekyros.ai/blog/vibe-coding-crisis-ai-technical-debt
    Missing preservation constraints as a source of scope creep: how refactoring sessions without explicit boundaries produce changes across unintended files and create technical debt at the PR level.

[^7]: GitClear — "2025 Coding on Copilot: 2023 Data Shows Downward Pressure on Code Quality," GitClear Research, 2025. https://www.gitclear.com/coding_on_copilot_data_shows_ais_downward_pressure_on_code_quality
    Unintended refactoring changes and their contribution to the codebase bloat pattern: how unbounded refactoring prompts produce more code additions than removals, the inverse of intended refactoring outcomes.

[^8]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
    Writer/reviewer pattern as the structural antidote to closed-loop review: fresh-context reviewer sessions and the specific issues they catch that implementing sessions cannot.

[^9]: Fannar Steinn Aðalsteinsson et al. — "Rethinking Code Review Workflows with LLM Assistance: An Empirical Study," arXiv:2505.16339, May 22, 2025. https://arxiv.org/abs/2505.16339
    Reviewer bias from shared context: how reviewers with knowledge of implementation choices produce less independent findings than reviewers evaluating code against specification.

[^10]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    Context flood failure mode: how injecting more context than necessary produces outputs that superficially acknowledge constraints without internalizing their specific implications for the task.

[^11]: Yonatan Sason — "The Black Box Problem: Why AI-Generated Code Stops Being Maintainable," *Towards Data Science*, March 6, 2026. https://towardsdatascience.com/the-black-box-problem-why-ai-generated-code-stops-being-maintainable/
    Context dilution: how important constraints buried in large context blocks receive less attention than structurally prominent constraints; the structural fix of leading with constraints rather than embedding them in documentation.

[^12]: Anthropic — "Security and Permissions," Claude Code Documentation, 2026. https://code.claude.com/docs/en/security-permissions
    Permission prompt mechanics: what information each prompt provides; the importance of reading before approving; sensitive-context permission restriction as a structural protection against reflexive approval.

[^13]: Sonar (SonarSource) — "Sonar Data Reveals Critical 'Verification Gap' in AI Coding," press release, January 8, 2026. https://www.sonarsource.com/company/press-releases/sonar-data-reveals-critical-verification-gap-in-ai-coding/
    Automation bias in permission approval: the 52% verification gap between stated distrust and actual verification behavior; the behavioral research on reflexive approval of AI actions.

[^14]: Anthropic — "Hooks Reference," Claude Code Documentation, 2026. https://code.claude.com/docs/en/hooks-reference
    PreToolUse audit logging as a compensation mechanism for reflexive approval: creating a session-level record of all tool calls and parameters that can be reviewed after the fact.

[^15]: Anthropic — "2026 Agentic Coding Trends Report," Anthropic, 2026. https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf
    Premature agentic delegation: the 0–20% fully delegatable task finding; the delegability checklist; supervised-by-exception as a safer alternative to fully autonomous sessions for new task types.

[^16]: METR — "We Are Changing Our Developer Productivity Experiment Design," METR Research, February 2026. https://www.metr.org/blog/2026-02-24-uplift-update/
    Productivity slowdown from premature delegation: the 19% slowdown on complex tasks as evidence that agentic delegation of the wrong task types produces more debugging overhead than it saves in execution time.

[^17]: Theo (t3.gg) — "Prompting AI Correctly Is a Skill and Most Devs Don't Have It," YouTube, January 2026. https://www.youtube.com/watch?v=9xRmNzGiBo4
    - Vague improvement requests in practice: live examples of the "make it better" anti-pattern and the specific ways the output diverges from the engineer's actual intent
    - Closed-loop review recognition: how to identify when you are asking for review from a non-independent context and the structural fix for each scenario
    - Permission skip behavior: why engineers reflexively approve permission prompts and the three-step habit change that restores meaningful permission review

[^18]: Fireship — "5 AI Coding Mistakes That Are Killing Your Productivity," YouTube, March 2026. https://www.youtube.com/watch?v=KpQ9R2HbfDg
    - The context flood anti-pattern: demonstration of how excessive context injection degrades output quality compared to minimum sufficient context prompts
    - Preservation constraint failure: before-and-after comparison of refactoring prompts with and without explicit "do not change" constraints
    - Premature delegation: real examples of agentic sessions that resolved implicit decision points incorrectly and the debugging cost that followed

[^19]: Sabrina Ramonov — "The ULTIMATE Claude Code Tutorial," YouTube, February 17, 2026. https://www.youtube.com/watch?v=fYX6hHC9FhQ
    - Anti-pattern catalog: all six anti-patterns demonstrated in context of a real development workflow, with structural alternatives shown in the same session
    - CLAUDE.md clarification instruction: how adding "ask before improving without specific objectives" to CLAUDE.md prevents the vague improvement request anti-pattern at the session level
    - Permission review practice: the specific habit that maintains meaningful permission oversight while minimizing friction in high-velocity sessions

[^a]: [Issues: Prompt Fragmentation](../Issues/07-prompt-fragmentation.md) — prompt anti-patterns are a primary cause of fragmentation; engineers who prompt poorly produce inconsistent output that compounds into the fragmentation risk described there.
[^b]: [Prompting: Prompt Architecture](01-prompt-architecture.md) — prompt anti-patterns are defined in contrast to prompt architecture; knowing what degrades output is the inverse of knowing what structures it well.
[^c]: [Governance: Review Policies](../Governance/01-review-policies.md) — reviewers should recognize when AI-generated code bears the signatures of anti-pattern prompting; the anti-pattern analysis informs what review should look for.