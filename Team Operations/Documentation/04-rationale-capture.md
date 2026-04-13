## Rationale Capture for AI-Generated Decisions

**Related to:** [Documentation Overview](00-overview.md) — Area 4: Rationale Capture for AI-Generated Decisions · [Documentation: Architecture Decision Records](01-architecture-decision-records.md)[^a] · [Issues: Comprehension Debt](../Issues/01-comprehension-debt.md)[^b] · [Tooling: CLAUDE.md Configuration](../Tooling & Configuration/01-claude-md-configuration.md)[^c] · [Workflows: Context Engineering](../Workflows/03-context-engineering.md)[^d]

---

## Overview

In pre-AI development, the rationale for a technical decision was typically recoverable even without documentation. The engineer who made it was usually still available to explain it, the decision process left traces in PR comments and Slack threads, and the code structure itself often made the reasoning apparent to a careful reader. Rationale documentation was valuable but not urgent — teams could function with significant rationale debt because the people who held the rationale were still present.

AI-assisted development breaks this assumption at scale. Claude Code makes implementation decisions in every session — selecting algorithms, choosing data structures, structuring error handling, naming things, laying out modules — and engineers who accept those decisions in review often do not examine or record the reasoning behind them. The engineer accepting the code was not present for the decision in the way a human collaborator would be; they reviewed the output, not the reasoning. The result is a specific class of documentation deficit: code that is functional and reviewed but whose structural choices cannot be explained by any team member.[^1]

This is the rationale gap, and it is distinct from standard documentation debt. Standard documentation debt means the team did not write down what they know. The rationale gap means the team accepted decisions that were made in a context — the AI session — that they did not fully inhabit, and that context is now closed. Retroactive reconstruction is possible in some cases; in others, the rationale is permanently inaccessible. The only effective solution is capturing rationale during the session, before the context closes.

---

## Section 1: The Rationale Gap Problem

**Description:** The rationale gap forms at a specific moment in the AI-assisted development workflow: when an engineer accepts AI-generated code without asking the AI to explain the decisions embedded in it. This acceptance is usually not negligent — the engineer reviews the code, runs the tests, evaluates the functionality, and makes a reasonable professional judgment that the output is correct. What they typically do not do is ask: "Why did you structure this as a recursive function rather than an iterative one?" "Why did you use a map here rather than a list of objects?" "Why did you handle the error this way rather than propagating it up?" These are the implementation decision questions whose answers exist in the model's inference process during the session and nowhere afterward.[^2]

The gap compounds because downstream code inherits unexplained decisions. A future session that modifies the component will see the existing structure and treat it as a constraint — "this is how this component works" — without knowing whether the structure was a deliberate architectural choice or an arbitrary AI default. That session may build on an unexplained convention that should have been changed, or may change something that was unexplained but actually essential. The inability to distinguish deliberate from arbitrary in AI-generated code is the fundamental challenge that rationale capture addresses.[^3]

**Recommended Practice:**
- Define the rationale gap explicitly in the team's engineering practices documentation: it is not a documentation failure but a session discipline failure. The rationale gap forms because the session ends without the engineer asking Claude to explain its key decisions. Closing the gap requires changing what happens in sessions, not only what happens after them.[^2]
- Identify the categories of AI decision that most commonly produce rationale gaps: algorithm selection, data structure choice, error handling strategy, module boundary placement, and API surface design. These are the decisions that are (a) non-obvious to a future reader and (b) most consequential when changed without understanding the original rationale.[^4]
- Track rationale gap incidents as a team metric: count the number of times in a given sprint that an engineer — in review, in a subsequent session, or during an incident — encountered AI-generated code that no one on the team could explain. This metric should decrease as the team's rationale capture discipline improves.
- Include the rationale gap in the team's onboarding to AI-assisted development: new engineers should understand why the gap forms, what it costs, and which session practices prevent it. An engineer who knows about the gap will ask the explanation question; an engineer who doesn't know about it won't know to ask.[^1]

---

## Section 2: Rationale Capture as a Session Discipline

**Description:** Rationale capture must happen during the session, not after. Once the session closes, the conversational context that produced the implementation decisions is gone — the engineer can reconstruct some rationale from the code itself, but the alternatives Claude considered and rejected, the tradeoffs it weighed, and the reasons it made specific choices are no longer accessible. The only way to capture this rationale is to ask for it before closing the session.[^6]

The session discipline is simple: before accepting a significant AI-generated implementation, ask Claude to explain its key decisions. The explanation should cover the choices that are not obvious from reading the code — the ones where a future engineer might reasonably wonder why this approach was taken. Claude's explanation of its own decisions is not authoritative (it is reconstructing rather than reporting), but it is substantially more informative than what can be inferred from the code alone, and it becomes the raw material for the rationale documentation that belongs in commit messages, PR descriptions, and CLAUDE.md.[^7]

**Recommended Practice:**
- Define a standard rationale elicitation prompt that engineers use before accepting significant AI-generated code: "Before I commit this, explain the key implementation decisions you made — the ones where you chose one approach over a reasonable alternative. For each decision, say what alternatives you considered and why you chose this approach." This prompt produces a rationale explanation that the engineer reviews and records.[^6]
- Calibrate when to ask: not every line of AI-generated code warrants rationale capture. The threshold is whether a future engineer would need to understand the reasoning to safely modify the code. The heuristic is: if you could imagine a future session accidentally undoing this decision without realizing it, document the rationale.[^3]
- Save the rationale explanation from the session as a text block that feeds directly into the PR description and commit message. The engineer edits it for length and accuracy — they do not rewrite it from scratch. This reduces the per-decision documentation cost to under three minutes and makes rationale capture economically sustainable as a session habit.[^8]
- Add rationale elicitation to the session closing checklist in CLAUDE.md: "Before ending any session that produces code for merge: (1) Ask Claude to explain key implementation decisions. (2) Save the explanation as the basis for the PR rationale field. (3) Transfer any architectural constraints to CLAUDE.md with the ADR reference." This makes the session closing checklist the enforcement mechanism for the discipline.[^9]

---

## Section 3: Embedding Rationale in Commit Messages and PR Descriptions

**Description:** Commit messages and PR descriptions are the most persistent rationale record in a codebase — they are versioned with the code, accessible from the Git history, and readable from Claude Code sessions via the repository context. A commit message that records not just what changed but why the change was made the way it was made is a durable rationale artifact that survives the session that produced it, survives the engineer who wrote it leaving the team, and survives the codebase evolving away from the patterns it describes.

Standard commit message practice produces messages that describe what changed: "Add session management service," "Refactor authentication handler," "Fix user query performance." These are useful for Git log navigation but useless for rationale recovery. A rationale-bearing commit message adds the why: "Add session management service using Redis for session storage — stateless service required for horizontal scaling planned in Q3; Redis chosen over in-memory store because session sharing across instances is needed; see ADR-008 for alternatives considered." The difference in writing time is under two minutes; the difference in maintenance value is substantial.

**Recommended Practice:**
- Adopt a commit message template for AI-primary commits: **[What]** [brief description of what changed]. **[Why]** [the key decision rationale, in one to three sentences]. **[ADR]** [ADR reference if applicable, otherwise "no ADR"]. The template takes 60–90 seconds to fill and produces a commit history that supports both Git navigation and rationale recovery.
- Define the PR description rationale section as a required field for AI-heavy PRs: "Key decisions and rationale: [explanation drawn from the session rationale elicitation, edited for clarity]." This field is not a duplicate of the commit message — it is the expanded rationale for the PR as a whole, providing context that individual commit messages may not carry.[^12]
- Train the team to treat a PR description without a rationale section as an incomplete PR, in the same way that a PR without tests is incomplete. The review process should explicitly check whether the rationale field is present and informative — not just whether the code looks correct. A correct but unexplained implementation is a knowledge debt generator.[^8]
- Use Claude Code to draft rationale sections for commits and PRs directly from the session's rationale elicitation output: "Draft a commit message rationale section from this session explanation: [paste Claude's explanation]. Format it as: 'Key decisions: [two to three sentences].' Keep it under 80 words." The draft will require light editing; the engineering cost is under a minute.[^7]

---

## Section 4: CLAUDE.md as a Rationale Repository

**Description:** CLAUDE.md serves two distinct functions: it encodes constraints for Claude Code sessions (what to do, what not to do, what patterns to follow), and it can serve as a rationale repository that connects those constraints to the decisions that produced them. Most teams use CLAUDE.md only for the first function — they add constraints without adding the reasoning, producing a system prompt that Claude follows but that engineers cannot interrogate. An engineer who joins a team with a well-populated CLAUDE.md but no rationale in it finds a list of rules without explanation — and a list of unexplained rules is one that future sessions will erode or contradict without recognizing it.[^13]

CLAUDE.md with rationale is substantially more useful for both humans and AI sessions. A constraint with rationale is not just "use the repository pattern for all database access" but "use the repository pattern — direct ORM use in service layer was producing untestable code in Q1; this constraint was established in ADR-003; do not suggest direct ORM access even where it would be simpler." The rationale-bearing constraint tells the session why the rule exists, making it much less likely that the session will suggest a well-intentioned exception that actually violates the intent of the constraint.[^9]

**Recommended Practice:**
- Audit the existing CLAUDE.md for constraints without rationale: for each constraint that lacks an explanation, add a brief rationale comment of one to two sentences. The retrospective rationale may require consulting ADRs, PR history, or asking the engineer who added the constraint. This is a one-time investment for existing constraints; all future constraints should include rationale at the time of addition.[^13]
- Establish a standard format for CLAUDE.md rationale comments: each constraint entry is followed by an inline comment beginning with `# Rationale:` that explains why the constraint exists, when it was established, and where to find more detail. The inline comment format makes the rationale visible to both humans reading the file and AI sessions processing it.[^9]
- Treat CLAUDE.md as the connection point between three documentation layers: ADRs (the full decision record), CLAUDE.md (the constraint with rationale summary and ADR reference), and commit messages (the per-change rationale record). These three layers together create a rationale infrastructure that a new engineer or a new session can navigate from any entry point — from the code, from Git history, from CLAUDE.md, or from the ADR record.[^14]
- When a CLAUDE.md constraint is changed or removed, require a rationale entry for the change: "Removed repository pattern requirement — ADR-003 superseded by ADR-012; direct ORM now acceptable in read-only query handlers following Q3 refactor." The change history of CLAUDE.md is as important as its current state — future sessions need to know not just what the current constraints are but why earlier constraints were retired.[^4]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Rationale Gap Tracking | Define and begin tracking rationale gap incidents as a sprint metric | Architect |
| Session Closing Checklist | Add rationale elicitation step to CLAUDE.md session closing checklist | Architect |
| Commit Message Template | Adopt What/Why/ADR commit message template for AI-primary commits | Architect |
| PR Rationale Field | Add rationale section to AI-heavy PR template; treat as required field in review | Architect |
| CLAUDE.md Rationale Audit | Audit existing constraints; add rationale comments in standard format | Architect |
| CLAUDE.md Change Rationale | Require rationale entry for all CLAUDE.md constraint changes or removals | Architect |

---

[^1]: Yue Liu et al. — "Debt Behind the AI Boom: A Large-Scale Empirical Study of AI-Generated Code in the Wild," arXiv:2603.28592, March 30, 2026. https://arxiv.org/html/2603.28592
 The rationale gap as a distinct category of AI-generated technical debt: how accepted-without-explanation decisions accumulate into codebases whose structural choices cannot be explained by any team member.

[^2]: Sreecharan Sankaranarayanan — "Mitigating 'Epistemic Debt' in Generative AI-Scaffolded Novice Programming using Metacognitive Scripts," arXiv:2602.20206, February 22, 2026. https://arxiv.org/abs/2602.20206
 Session discipline as the primary prevention for the rationale gap: the metacognitive prompting practices that force explicit reasoning before acceptance; the mechanism by which epistemic debt forms at the acceptance moment.

[^3]: Kyros — "The Vibe Coding Crisis: How AI-Generated Technical Debt Is Costing Companies Millions," March 2026. https://usekyros.ai/blog/vibe-coding-crisis-ai-technical-debt
 Rationale gap compounding: how downstream code inherits unexplained decisions and treats them as constraints, amplifying the original gap; the inability to distinguish deliberate from arbitrary in AI-generated code.

[^4]: DEV Community — "AI Is Creating a New Kind of Tech Debt — And Nobody Is Talking About It," March 2026. https://dev.to/harsh2644/ai-is-creating-a-new-kind-of-tech-debt-and-nobody-is-talking-about-it-3pm6
 Categories of AI implementation decision most prone to rationale gaps: algorithm selection, data structure choice, error handling strategy; the CLAUDE.md change rationale requirement.

[^6]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
 Rationale capture as a session-closing discipline: the standard elicitation prompt and how it produces documentation that post-hoc reconstruction cannot replicate; the economic case for capturing rationale before the session closes.

[^7]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
 Using Claude Code to draft rationale documentation from session explanations: how to transform a session's rationale elicitation output into commit message and PR description content with minimal engineering overhead.

[^8]: Fannar Steinn Aðalsteinsson et al. — "Rethinking Code Review Workflows with LLM Assistance: An Empirical Study," arXiv:2505.16339, May 22, 2025. https://arxiv.org/abs/2505.16339
 Rationale field as a required PR element: treating a correct-but-unexplained implementation as an incomplete PR; the review discipline that makes rationale documentation a merge criterion rather than a documentation preference.

[^9]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
 CLAUDE.md as rationale repository: the inline rationale comment format; the session closing checklist structure; how CLAUDE.md rationale-bearing constraints are more resistant to AI session erosion than constraint-only entries.

[^12]: CodeRabbit — "State of AI Code Generation: AI vs. Human Code Report," December 17, 2025. https://www.coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report
 PR description rationale sections: how expanded rationale in PR descriptions provides context that commit-level messages cannot carry; the relationship between PR rationale quality and review effectiveness.

[^13]: Dex Horthy (YC Root Access) — "Advanced Context Engineering for Agents," YouTube, August 2025. https://www.youtube.com/watch?v=IS_y40zY-hc
 CLAUDE.md as a rationale repository: how rationale-bearing constraints are more effective session constraints than rule-only entries; the audit practice for adding rationale to existing constraints.

[^14]: Anthropic — "Model Context Protocol," Anthropic, 2025. https://www.anthropic.com/news/model-context-protocol
 Three-layer rationale infrastructure: ADRs, CLAUDE.md, and commit messages as complementary rationale stores that support navigation from any entry point; MCP as the mechanism connecting external rationale stores to session context.

[^a]: [Documentation: Architecture Decision Records](01-architecture-decision-records.md) — ADRs capture formal architectural decisions; rationale capture extends this to informal AI-generated decisions below the ADR threshold; the two practices together close the full documentation gap.
[^b]: [Issues: Comprehension Debt](../Issues/01-comprehension-debt.md) — comprehension debt is the direct consequence of failed rationale capture; undocumented AI decisions are the primary source of debt accumulation.
[^c]: [Tooling: CLAUDE.md Configuration](../Tooling & Configuration/01-claude-md-configuration.md) — CLAUDE.md entries often originate from rationale capture; the decision to constrain a behavior is itself an AI-generated decision that requires documentation.
[^d]: [Workflows: Context Engineering](../Workflows/03-context-engineering.md) — rationale capture feeds back into context engineering; documented decisions become the session context that prevents the same ground from being rediscovered.
