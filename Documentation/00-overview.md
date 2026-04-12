## Overview

For a team of 11 — 4 backend engineers, 3 frontend engineers, 1 architect, 1 QA engineer, and 2 product managers — integrating Claude Code into daily workflows, the documentation infrastructure supporting the codebase faces a challenge that did not exist in pre-AI development: the gap between what the codebase contains and what the team understands is now structurally wider, closes more slowly, and is harder to audit. When engineers wrote every line of code themselves, the codebase and the team's knowledge of it diverged at a pace bounded by human output. When AI generates significant portions of the codebase, the divergence can be orders of magnitude faster — and the standard documentation practices designed for human-paced development are insufficient to close it.[^1]

This documentation gap is not primarily about code comments or README files. It is about the three classes of knowledge that a functioning team needs and that AI-assisted development erodes: architectural decisions (what was decided and why), operational procedures (how to run, debug, and recover the system), and rationale (why the code is shaped the way it is, including the implicit decisions that AI made and engineers accepted without recording). A team that cannot answer these questions from its documentation is a team that will rediscover the same ground repeatedly — via production incidents, new engineer onboarding failures, and AI sessions that regenerate code that should have been kept, or keep patterns that should have been changed.[^2]

Four documentation areas are covered below. They are ordered by immediacy: ADRs prevent architectural knowledge loss at the decision point; runbook standards prevent operational knowledge loss before incidents; knowledge transfer practices prevent comprehension debt from compounding as teams and codebases evolve; rationale capture prevents the rationale gap from forming in the first place.

---

## Area 1: Architecture Decision Records (ADRs)

**Description:** Architecture Decision Records document significant decisions about the structure, behavior, and conventions of a system — not what the code does, but why it was shaped the way it is. In pre-AI development, this documentation was valuable but optional: engineers who made the decisions were typically still available to explain them, and the decision-making process itself created shared understanding. In AI-assisted development, ADRs become essential. AI sessions operate on present context without knowledge of decisions made in past sessions. Without ADRs, each Claude Code session risks undoing, contradicting, or ignoring decisions that were made deliberately — and the engineers who accepted AI-generated changes may not have deeply understood what was changed.[^3]

The ADR gap is not a failure of documentation discipline alone. It is a failure of the system: when decisions are made and implemented at AI speed, the lag between decision and documentation is longer, and the probability that documentation never happens is higher. ADRs for AI-assisted teams must be easier to produce, must be accessible from AI sessions, and must include a field that human documentation of past eras did not need: "what Claude should never change about this."[^4]

**Proposed Solution:**
- Adopt a lightweight ADR format specifically designed for AI-assisted teams: status, context, decision, constraints for AI sessions, and consequences. The constraints field makes ADRs actionable within Claude Code sessions, not just readable by humans.[^5]
- Store ADRs in a Google Drive folder and configure the Google Drive MCP server to make them accessible to Claude Code sessions, so every session can query the decision record before generating code in an affected area.[^6]
- Use Claude Code to draft ADRs from discussion transcripts, architecture meeting notes, and PR comment threads — then have the architect review and finalize. This removes the documentation lag by making ADR creation part of the AI session rather than a separate discipline.[^3]
- Require an ADR reference or a new ADR for any AI-primary PR that introduces a new pattern, changes a significant architectural convention, or modifies a component covered by an existing ADR. The ADR link in the PR is the mechanism that closes the loop between decision and documentation.[^4]

---

## Area 2: Runbook Standards

**Description:** Runbooks document operational procedures — how to deploy, how to debug common failure modes, how to recover from specific incidents, how to execute routine operational tasks. Standard runbook practice assumes that a human engineer reads the runbook and performs the steps with judgment. Claude Code changes this assumption: runbooks can now serve as machine-parseable execution plans that Claude Code reads and performs — and the step-by-step structure that makes runbooks human-readable also makes them AI-executable, but only if they meet specific formatting and precision requirements.[^7]

The runbook maintenance challenge in AI-assisted teams is that AI-generated code changes operational procedures at a pace that outstrips manual runbook updates. A runbook written for a human-authored deployment pipeline may be partially or entirely wrong for an AI-generated replacement — and the team may not discover this until a production incident requires following the outdated runbook under time pressure. Runbook currency is not a documentation nice-to-have; it is an operational safety requirement.[^8]

**Proposed Solution:**
- Adopt a runbook format with numbered steps, explicit preconditions, expected outputs, and failure paths. This structure is both maximally human-readable and Claude Code-executable — steps that are too ambiguous for AI execution are also too ambiguous for a new engineer following the runbook under pressure.[^7]
- Define an explicit runbook execution pattern for Claude Code: an engineer opens a Claude Code session, loads the runbook via Google Drive MCP, and issues a structured command to execute the procedure. The session reads each step, confirms preconditions, executes, and reports the result before proceeding. This is faster than manual execution and produces a full execution log.[^9]
- Require runbook review as part of the PR merge criteria for any AI-primary PR that modifies deployment, data migration, infrastructure configuration, or other operationally significant components. The PR author attests that affected runbooks have been updated or that no runbooks are affected.[^8]
- Store all runbooks in a dedicated Google Drive folder accessible via MCP, with a naming convention that makes them discoverable by topic: `runbook-deployment-production.md`, `runbook-database-migration.md`, `runbook-incident-authentication.md`. Discoverability is an operational requirement — a runbook that cannot be found during an incident has the same operational value as one that does not exist.[^6]

---

## Area 3: Knowledge Transfer in AI-Assisted Teams

**Description:** Traditional engineer onboarding assumed that an engineer who could read the code could understand the codebase. This assumption broke down slowly in pre-AI development — complex codebases always had dark corners — but it was approximately true for well-maintained codebases. In AI-assisted development, it is no longer approximately true. A codebase that has been heavily modified by AI sessions contains patterns that were generated without any engineer fully understanding them, accepted via review processes designed to verify function rather than ensure comprehension, and integrated without the shared understanding that comes from building something together.[^10]

The knowledge transfer failure mode is not individual; it is structural. An engineer onboarding to an AI-heavy codebase cannot simply read the code to understand it, because the code does not contain the history of why it is shaped the way it is — the session transcripts, the alternatives Claude considered and rejected, the prompts that produced specific structures. Without deliberate knowledge transfer practices, this comprehension debt accumulates and compounds: each new session adds more context that only exists in past sessions, each new engineer inherits less understandable code, and the team's collective ability to maintain and extend the system degrades faster than standard technical debt metrics capture.[^11]

**Proposed Solution:**
- Require a comprehension summary as part of the merge criteria for AI-heavy PRs: a brief explanation by the PR author of what the AI-generated code does, why it is structured as it is, and what the key tradeoffs were. This summary is written for the next engineer — the onboarder who will inherit this code without having been present for the session.[^12]
- Treat session transcripts for significant architecture or implementation sessions as a documentation artifact: store them in Google Drive alongside ADRs and runbooks, linked from the relevant PR. Not all sessions warrant archiving — the criterion is whether a future engineer would benefit from reading the history of how this code came to be.[^10]
- Design the new engineer onboarding process for AI-assisted codebases explicitly: the reading list is not "read the codebase" but "read the ADRs, read the session transcripts for the components you'll own, read the rationale captures in CLAUDE.md, and then read the code." This order produces comprehension that the code-first approach no longer provides.[^11]
- Measure comprehension as an explicit team health metric, not a proxy metric. The Questions Without Answers metric — the number of questions a new engineer asked about the codebase that no existing engineer could answer — is a direct measure of comprehension debt that standard velocity and coverage metrics do not capture.[^13]

---

## Area 4: Rationale Capture for AI-Generated Decisions

**Description:** Rationale capture is the practice of recording not just what was decided but why — including decisions that AI made during implementation that engineers accepted without consciously deciding. This class of decision is new in AI-assisted development. Before AI, every line of code reflected a human engineer's implicit choices about structure, naming, algorithms, error handling, and patterns — choices that were often undocumented but at least knowable by asking the author. When AI generates code and an engineer approves it without examining every implicit choice, the rationale for those choices exists only within the AI session that is now closed.[^14]

The rationale gap compounds over time. As AI-generated code is built upon by subsequent AI sessions, each of which inherits implicit decisions without understanding their provenance, the gap between the codebase's shape and the team's ability to explain that shape widens. Teams that have experienced this describe a specific symptom: engineers who are afraid to change things because they don't know why they are the way they are — a form of comprehension debt that manifests as velocity loss and risk aversion rather than visible technical debt.[^15]

**Proposed Solution:**
- Establish rationale capture as a session discipline: before committing AI-generated code, ask Claude to explain the key implementation choices it made — the algorithm selected, the data structure chosen, the error handling approach, the naming conventions. This explanation becomes the rationale record.[^16]
- Embed rationale in commit messages and PR descriptions as a standard template field: "Key decisions: [what AI decided and why, in plain language]." A commit message that says "Add user session management" captures what was done; one that also says "Used Redis over in-memory store because session sharing across server instances is required for the planned horizontal scaling" captures why — and that why is what future sessions need.[^14]
- Maintain CLAUDE.md as a rationale repository alongside a constraint repository. For each significant constraint or convention in CLAUDE.md, add a rationale comment: not just "Use the repository pattern for all database access" but "Use the repository pattern — decision rationale in docs/adr/003-repository-pattern.md — because direct ORM use in service layer was producing untestable code in Q1." This makes CLAUDE.md educationally useful rather than just operationally constraining.[^5]
- Define a rationale capture threshold: not every AI decision warrants documentation, but decisions that affect the public API, the data model, cross-service interfaces, security boundaries, or the team's established conventions do. The architect sets and maintains this threshold; the team applies it as part of the PR review checklist.[^15]

---

## Summary of Recommended Actions

| Documentation Area | Immediate Action | Owner |
|---|---|---|
| Architecture Decision Records | Adopt lightweight AI-team ADR format; store in Google Drive; configure MCP access | Architect |
| Runbook Standards | Adopt machine-parseable runbook format; store in Google Drive; define Claude Code execution pattern | Backend lead |
| Knowledge Transfer | Add comprehension summary to AI-heavy PR merge criteria; redesign onboarding reading list | Architect |
| Rationale Capture | Add rationale session discipline to PR checklist; add rationale comments to CLAUDE.md entries | Architect |

---

[^1]: DEV Community — "AI Is Creating a New Kind of Tech Debt — And Nobody Is Talking About It," March 2026. https://dev.to/harsh2644/ai-is-creating-a-new-kind-of-tech-debt-and-nobody-is-talking-about-it-3pm6
    The structural documentation gap in AI-assisted development: how AI-speed output creates knowledge divergence that documentation practices designed for human-paced development cannot close.

[^2]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    The three classes of knowledge that AI-assisted development erodes: architectural decisions, operational procedures, and implementation rationale. The documentation infrastructure that prevents each class of erosion.

[^3]: Michael Nygard — "Documenting Architecture Decisions," Cognitect, November 2011. https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions
    Original ADR format and rationale; the foundational case for why decisions rather than just designs need documentation; the template that most ADR tooling is derived from.

[^4]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    CLAUDE.md as the mechanism for encoding architectural decisions into every session: how ADR constraints in the system prompt prevent AI sessions from contradicting documented decisions.

[^5]: Fannar Steinn Aðalsteinsson et al. — "Rethinking Code Review Workflows with LLM Assistance: An Empirical Study," arXiv:2505.16339, May 22, 2025. https://arxiv.org/abs/2505.16339
    ADR linkage in PR review: how connecting AI-generated code to architectural decisions creates the accountability record that review processes need to evaluate whether AI output is architecturally coherent.

[^6]: Anthropic — "Model Context Protocol," Anthropic, 2025. https://www.anthropic.com/news/model-context-protocol
    MCP architecture for connecting Claude Code sessions to external documentation stores: how Google Drive MCP enables AI sessions to query ADRs, runbooks, and rationale documentation at session time.

[^7]: Ravikanth Konda — "Human-AI Collaboration in Software Teams: Evaluating Productivity, Quality, and Knowledge Transfer with Agentic and LLM-Based Tools," *International Journal of AI, BigData, Computational and Management Studies*, February 17, 2026. https://ijaibdcms.org/index.php/ijaibdcms/article/view/418
    Runbook standards for AI-assisted operations: the precision requirements that make runbooks both human-readable and machine-executable; operational knowledge transfer in AI-heavy teams.

[^8]: GitClear — "Coding on Copilot: 2025 Data Suggests Downward Pressure on Code Quality," GitClear Research, 2025. https://www.gitclear.com/coding_on_copilot_data_suggests_coding_quality_is_decreasing
    Runbook currency as an operational safety requirement: how AI-generated code changes operational procedures at a pace that outstrips manual documentation update practices.

[^9]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
    Claude Code as an execution agent for structured procedures: how agentic session patterns apply to runbook execution and the session discipline required for safe automated operational steps.

[^10]: Sreecharan Sankaranarayanan — "Mitigating 'Epistemic Debt' in Generative AI-Scaffolded Novice Programming using Metacognitive Scripts," arXiv:2602.20206, February 22, 2026. https://arxiv.org/abs/2602.20206
    Epistemic debt and its relationship to AI-assisted codebase maintenance: how comprehension debt accumulates structurally rather than individually, and the documentation practices that prevent it.

[^11]: Judy Hanwen Shen and Alex Tamkin (Anthropic) — "How AI Assistance Impacts the Formation of Coding Skills," arXiv:2601.20245, January 28, 2026. https://arxiv.org/abs/2601.20245
    Comprehension gaps in AI-assisted teams: the structural divergence between code complexity and team understanding; how onboarding fails when the codebase contains knowledge that only exists in closed sessions.

[^12]: CodeRabbit — "State of AI Code Generation: AI vs. Human Code Report," December 17, 2025. https://www.coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report
    Comprehension documentation as a merge criterion: the specific fields that make AI-heavy PR reviews produce knowledge artifacts rather than only quality gates.

[^13]: Gartner — "Predicts 2026: Software Engineering and DevSecOps," Gartner Research, January 2026. https://www.gartner.com/en/documents/predicts-2026-software-engineering-devsecops
    Comprehension metrics in AI-assisted engineering teams: the questions-without-answers indicator and its relationship to team capability degradation over time.

[^14]: The Pragmatic Engineer — "AI Tooling for Software Engineers in 2026," March 2026. https://newsletter.pragmaticengineer.com/p/ai-tooling-2026
    The rationale gap and its compounding effect: how AI sessions that inherit undocumented decisions produce codebases shaped by reasons that no engineer can reconstruct.

[^15]: Kyros — "The Vibe Coding Crisis: How AI-Generated Technical Debt Is Costing Companies Millions," March 2026. https://usekyros.ai/blog/vibe-coding-crisis-ai-technical-debt
    Rationale debt as a velocity risk: the specific mechanism by which comprehension gaps manifest as risk aversion and reluctance to change — the "afraid to touch it" failure mode in AI-heavy codebases.

[^16]: Yue Liu et al. — "Debt Behind the AI Boom: A Large-Scale Empirical Study of AI-Generated Code in the Wild," arXiv:2603.28592, March 30, 2026. https://arxiv.org/html/2603.28592
    Rationale capture as a session discipline: the before-commit explanation request and how it produces documentation that post-hoc reconstruction cannot replicate.
