## Overview

Agentic delegation is the practice of handing tasks to Claude Code with the expectation that it will complete them autonomously — without interactive steering, without mid-session clarification, and without the engineer watching the terminal. It represents the highest-trust, highest-leverage mode of AI-assisted development, and also the highest-risk one if applied without discipline.

The current state of agentic capability is well-characterized by Anthropic's 2026 research: developers can "fully delegate" only 0–20% of tasks without supervision. The remaining 80–100% require what the report calls "constant collaboration requiring supervision, validation, and human judgment."[^1] This is not a capability failure; it is an accurate description of the current boundary between what AI can own reliably and what requires human involvement. The teams creating the most value from agentic delegation are those who have identified their 20% accurately and developed systematic practices around it.

This memo covers the theoretical and practical constraints on delegation, the criteria for identifying delegable tasks, fan-out patterns for large-scale migrations, bounded autonomy governance, and the accountability frameworks that apply when delegation goes wrong.

---

## Section 1: The 0–20% Ceiling and Its Origins

**Description:** Anthropic's 2026 Agentic Coding Trends Report found that developers report being able to fully delegate 0–20% of their tasks — with the wide range reflecting significant variation across engineers, codebases, and task types.[^1] This ceiling is not arbitrary. It reflects the intersection of three constraints: task specificity (how completely can requirements be specified without organizational context?), verification completeness (does a pass/fail signal exist for every relevant dimension?), and reversibility (if the agent's output is wrong, how costly is correction?).

Tasks at the low end of delegation success share identifiable characteristics: underspecified requirements, absent verification mechanisms, high organizational context dependency, and significant blast radius if wrong. Tasks at the high end share the opposite: requirements that are fully specifiable in text, automated verification available, low organizational context dependency, and easy reversal or review before merge.[^2]

The academic framework for intelligent delegation — from a February 2026 Google DeepMind paper — frames this as a contingency-based decision problem: low-stakes tasks (low criticality, high reversibility, clear objectives) support AI delegation with minimal oversight; high-stakes tasks (critical outcomes, irreversible effects, subjective success criteria) require mandatory human approval.[^3] "Automation is therefore not only about what AI can do, but what AI should do."[^3]

CIO's 2026 analysis of how agentic AI reshapes engineering workflows describes the emerging three-stage model: assistance (AI supports discrete atomic tasks), augmentation (AI manages multi-step workflows within bounded domains), and autonomy (AI operates across domains).[^4] Most teams in 2026 are operating in the assistance-to-augmentation transition. Full autonomy for complex engineering work remains ahead. Treating current tools as operating at the autonomy stage when they are at the augmentation stage is the primary source of costly delegation failures.

**Recommended Practice:**
- Track delegation outcomes at the sprint level: which tasks were delegated, which succeeded without steering, which required significant correction. Use this data to calibrate the team's delegable 20% accurately rather than estimating.[^1]
- Brief the team on the 0–20% ceiling explicitly. Delegation expectations calibrated to realistic capability prevent the overpromising that generates review theater and comprehension debt.[^5]
- When evaluating a potential delegation task, ask: can requirements be fully specified without a conversation? Is there a pass/fail verification signal? Would incorrect output be caught before merge? Three "yes" answers suggest delegability.[^3]
- Review delegation failure patterns quarterly: which task types failed more often than expected? Adjust the team's delegable task list accordingly.[^2]

---

## Section 2: The Delegability Criteria

**Description:** Identifying which tasks are in the delegable 20% requires a consistent evaluation framework. The criteria that reliably separate delegable from non-delegable tasks cluster into five dimensions:

**Requirement completeness.** Delegable tasks have requirements that can be written down completely in a text prompt. If completing the task requires knowledge that is organizational, relational, or contextual in ways that cannot be expressed as text (what the customer actually wants, what the team decided in last week's meeting, what the undocumented reason is for a particular design choice), the task cannot be reliably delegated.[^2]

**Verification availability.** Delegable tasks have a pass/fail verification mechanism Claude can run: a test suite, a build, a linter, a screenshot comparison, a dry-run output. Without this, Claude cannot self-correct, and incorrect output reaches human review unfiltered.[^6]

**Scope boundedness.** Delegable tasks touch a well-understood, narrow scope. Unbounded scope — "improve the authentication system," "optimize the database layer" — produces AI output that is architecturally incoherent because there is no principled basis for Claude's scope decisions.[^2]

**Pattern precedence.** Delegable tasks follow patterns that already exist in the codebase. A new REST endpoint that mirrors the structure of ten existing ones can be delegated confidently; a first-of-its-kind architectural pattern cannot.[^7]

**Reversibility.** Delegable tasks produce output that is reviewed before any irreversible action — no direct deployment, no database migration applied without review, no external API called during generation.[^3]

**Recommended Practice:**
- Use the five-criteria checklist as a sprint-planning gate before scheduling AI-primary stories: score each criterion as met or not met. Stories failing two or more criteria should be re-scoped or reclassified as AI-assisted rather than AI-primary.[^2]
- Establish the reversibility criterion as a hard gate: any task that involves an irreversible action (database migration, infrastructure change, external API write) requires human review before execution regardless of other criteria.[^3]
- Document the team's delegable task library: which task types consistently pass all five criteria? This library grows over sprints and becomes a team knowledge asset.[^7]
- Assign the architect to review the delegable task library quarterly: which new task types have become delegable as the codebase matures and test coverage improves?[^5]

---

## Section 3: Fan-Out Patterns for Large-Scale Migrations

**Description:** The highest-value use case for agentic delegation is large-scale, repetitive migrations: renaming a pattern across hundreds of files, updating an API version across all callers, converting a test suite to a new framework, applying a security fix to all instances of a vulnerability pattern. These tasks are ideal delegation candidates because they have high pattern precedence, bounded scope per instance, and fast verification (does the build pass? do the tests pass?).[^8]

The fan-out pattern is the operational approach: generate a task list, write a loop script using `claude -p`, test on a small sample before scaling, and use `--allowedTools` to scope what Claude can do in each unattended run.[^8]

The step-by-step pattern:
1. Generate the task list: "List all files that contain the deprecated `fetchUser()` call pattern."
2. Write a loop script: `for file in $(cat files.txt); do claude -p "Update $file to replace fetchUser() with getUserById(). Return OK or FAIL." --allowedTools "Edit,Bash(git add *,git commit *)"; done`
3. Test on three files first. Review the output. Refine the prompt if needed.
4. Run at scale. Monitor for FAIL responses and address them separately.
5. Run the full test suite across all changed files before merging.[^8]

The incident.io team's experience with this pattern produced an 18% improvement (30-second reduction) in API generation time for a $8 Claude credits investment — because the task met all delegability criteria: repetitive pattern, bounded scope, fast verification (build + test), fully specifiable requirement.[^9]

**Recommended Practice:**
- Use fan-out patterns for migrations, security patch applications, dependency updates, and API version changes — tasks that are repetitive, bounded, and verifiable.[^8]
- Always test the prompt on 2-3 instances before running at scale. Prompts that fail on small samples will fail proportionally at scale. Refine before scaling.[^8]
- Use `--allowedTools` to scope permissions for unattended runs. Specify exactly which tools and commands Claude may use; do not run fan-out patterns with unrestricted tool access.[^8]
- Monitor fan-out runs with `--output-format stream-json` and log FAIL responses to a separate file. Review failures before committing the successful batch.[^8]

---

## Section 4: Bounded Autonomy in Practice

**Description:** "Bounded autonomy" is the leading governance model for agentic AI in 2026: giving agents clear operational limits, mandatory escalation paths to humans for high-stakes decisions, and comprehensive audit trails of what the agent did and why.[^10] It is the middle path between full human-in-the-loop (interactive steering of every step) and full autonomy (no human involvement until output is produced).

The bounds are practical, not philosophical: scope limits (which files may the agent touch?), tool limits (which commands may the agent run?), time limits (how long may the agent operate before checking in?), and cost limits (how much may the agent spend before pausing?). Each bound is a safety mechanism that limits blast radius if the agent's understanding of the task diverges from the engineer's intent.[^3]

Claude Code provides bounded autonomy through several mechanisms: `--allowedTools` scopes what actions are permitted; auto mode provides a background safety classifier that blocks scope escalation; permissions settings allowlist specific safe commands; sandbox mode restricts filesystem and network access.[^8] For unattended runs, all of these should be configured explicitly rather than relying on defaults.

Google DeepMind's February 2026 framework for intelligent delegation warns specifically about long delegation chains (A delegates to B, B delegates to C). Each step in the chain creates an accountability gap where the original human intent becomes disconnected from execution.[^3] For our team, this applies to multi-agent orchestration scenarios: when an orchestrator delegates to subagents who may further delegate, the accountability chain must be explicitly defined, not implicit.

**Recommended Practice:**
- Configure explicit bounds for every unattended agent run: allowed tools, scope of files, maximum runtime, and maximum spend.[^8]
- Use auto mode with `--permission-mode auto` for tasks where broad tool access is needed but active supervision is not. The background safety classifier blocks scope escalation while allowing routine work to proceed.[^8]
- For multi-agent orchestration, define an explicit accountability map: who owns the orchestrator session's output? Which subagents can take write actions? What requires human approval before execution?[^3]
- Log all unattended agent runs with `--output-format json` to a persistent audit trail. If a delegation produces unexpected output, the audit log is the primary diagnostic tool.[^8]

---

## Section 5: Accountability in Delegation Chains

**Description:** Delegation reduces effort, not accountability. The engineer who delegates a task to an agent is accountable for the output of that task — including output the agent produced without human oversight. This is not merely a philosophical position; it is the practical basis of team trust and codebase quality. If delegation removes accountability, it creates a diffuse responsibility structure where no one owns AI-generated errors — precisely the dynamic that produces the comprehension debt and security vulnerabilities documented in the Issues section.[^5]

The CIO's "Delegate, Review, Own" model captures the accountability structure precisely: AI agents handle first-pass execution; engineers review outputs for correctness and risk; humans retain ownership of architecture, tradeoffs, and outcomes.[^4] The review step is non-negotiable. Delegation without review is not a workflow pattern — it is an abdication of the engineering responsibility that justifies employment.

The Google DeepMind "liability firebreak" concept is practically relevant for complex multi-agent workflows: at critical decision points in a delegation chain, an intermediary agent must either assume full accountability for continuing or halt and request renewed human authorization. This prevents the accountability vacuum where complex agent output exists in a space where no one — human or agent — clearly owns it.[^3]

For our team, the practical expression of this principle is the "review before merge" requirement: any output from an unattended agent session must pass the fresh-context reviewer session and the human code review before merging, regardless of whether tests pass. Passing tests is a necessary but not sufficient condition for merge.[^7]

**Recommended Practice:**
- Establish a team norm with explicit CTO endorsement: delegating a task to an agent does not transfer responsibility for the output. The delegating engineer owns the output.[^5]
- Require the full review pipeline (fresh-context reviewer session + human review) for all unattended agent output before merge. Test passage alone is insufficient.[^7]
- For multi-agent workflows, identify the liability firebreak points: which decisions require renewed human authorization before the delegation chain continues?[^3]
- Track delegation incidents (agent output that required significant correction, caused regressions, or introduced security issues) in retrospectives. These incidents are calibration data for the delegable task library, not individual failures to suppress.[^1]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Delegation Ceiling Calibration | Track delegation outcomes; adjust team's delegable task list each sprint | Architect |
| Delegability Criteria | Apply five-criteria checklist in sprint planning; hard gate on reversibility | Architect + engineers |
| Fan-Out Patterns | Test prompts on 2-3 instances before scaling; use --allowedTools for scope | All engineers |
| Bounded Autonomy | Configure explicit bounds (tools, scope, time, cost) for all unattended runs | All engineers |
| Accountability | "Delegate, Review, Own": delegating engineer owns the output; review before merge required | CTO + engineers |

---

[^1]: Anthropic — "2026 Agentic Coding Trends Report," Anthropic, 2026. https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf
    - 0–20% fully-delegable task ceiling: empirical finding across developers using AI coding agents
    - 80–100% of tasks require "constant collaboration requiring supervision, validation, and human judgment"
    - Real-world benchmarks: Rakuten 7-hour autonomous vLLM implementation; TELUS 30% faster code shipping

[^2]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    - Characteristics of delegable tasks: requirement completeness, bounded scope, high-precedent patterns
    - Execution loops vs. interactive prompting: how agentic workflows differ from pair programming
    - Human role shift: from writing code to defining intent, reviewing output, owning outcomes

[^3]: Google DeepMind — "Intelligent AI Delegation," arXiv, February 2026. https://arxiv.org/html/2602.11865v1
    - Contingency-based delegation criteria: low-stakes (delegate) vs. high-stakes (human approval)
    - Liability firebreak concept: intermediary agents must either assume accountability or halt for human reauthorization
    - "Automation is not only about what AI can do, but what AI should do"

[^4]: CIO — "How Agentic AI Will Reshape Engineering Workflows in 2026," April 2026. https://www.cio.com/article/4134741/how-agentic-ai-will-reshape-engineering-workflows-in-2026.html
    - "Delegate, Review, Own" operating model: AI executes, engineers review, humans retain ownership
    - Three-stage evolution: assistance → augmentation → autonomy
    - McKinsey data: AI-centric organizations achieving 20–40% operating cost reduction

[^5]: Issues Overview — "Issue 1: Comprehension Debt" and "Issue 8: The Velocity–Governance Feedback Loop," ClaudeCodeReview. Issues/overview.md
    - Accountability structures in AI-assisted teams: who owns AI-generated output?
    - CTO briefing requirement: delegation reduces effort, not accountability
    - Velocity–governance loop: how undisciplined delegation accelerates quality degradation

[^6]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    - Verification as prerequisite for delegation: pass/fail signal required before unattended run
    - Auto mode and safety classifier: blocking scope escalation in unattended runs
    - --allowedTools and permission allowlists for bounded autonomy

[^7]: Phillip Carter — "How I Code With LLMs These Days," Honeycomb, March 2025. https://www.honeycomb.io/blog/how-i-code-with-llms-these-days
    - Pattern precedence as a delegability criterion: existing patterns vs. first-of-kind architectural work
    - Agent limitations in existing codebases: "wandering into nonsensical code" without firm scope constraints
    - Incremental generation principle applied to delegation: small, verifiable tasks not sprawling features

[^8]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    - Fan-out pattern: generate task list → loop script with claude -p → test on 2-3 instances → run at scale
    - --allowedTools scoping for unattended runs; --output-format json for audit logging
    - Auto mode configuration; sandbox mode for filesystem and network restriction

[^9]: incident.io — "How We're Shipping Faster with Claude Code and Git Worktrees," incident.io Blog, 2026. https://incident.io/blog/shipping-faster-with-claude-code-and-git-worktrees
    - $8 Claude credits → 18% API generation improvement: fan-out delegation ROI at team scale
    - Bounded delegation criteria met: repetitive pattern, bounded scope, fast verification
    - Practical outcome measurement for delegation investment decisions

[^10]: Bunnyshell — "Agentic Development: What It Means for Engineering Infrastructure in 2026," 2026. https://www.bunnyshell.com/guides/agentic-development/
    - Bounded autonomy as the leading governance model: operational limits, escalation paths, audit trails
    - Infrastructure implications of agentic development: CI/CD adaptation, environment isolation, cost management
    - Human-in-the-loop design patterns for teams transitioning from interactive to agentic AI

[^11]: Tessl — "8 Agentic Coding Trends Shaping Software Engineering in 2026," March 2026. https://tessl.io/blog/8-trends-shaping-software-engineering-in-2026-according-to-anthropics-agentic-coding-report/
    - Multi-agent coordination: specialized agents in parallel under an orchestrator
    - Fan-out patterns for large codebases: distributing work across parallel Claude invocations
    - Security considerations for agentic coding: same tools that enable defensive work lower barriers for misuse

[^12]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
    - /batch and /schedule commands for extended autonomous delegation workflows
    - Plan-first before auto-accept: the sequence that enables high-quality unattended runs
    - Building for future model capability: designing delegation workflows for models six months ahead
