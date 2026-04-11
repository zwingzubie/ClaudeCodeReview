## Overview

For a team of 11 — including 4 backend engineers, 3 frontend engineers, 1 architect, 1 QA engineer, and 2 product managers — using Claude Code daily, having deliberate workflow practices matters as much as any policy around the risks of AI-generated code. Workflow discipline is where the promise of AI-assisted development either compounds or degrades: engineers who adopt structured patterns consistently report better output quality and fewer correction cycles, while those who treat Claude Code as interactive chat produce inconsistent results that tend to accumulate exactly the issues described in the Issues section.

The framing here is not about restricting AI use — it is about channeling it. Anthropic's own internal teams, where approximately 90% of Claude Code's source code is now written by Claude Code itself, use a documented set of workflow practices that make this sustainable.[^1] Those patterns are not proprietary. They are well-documented and directly applicable to a team of this size.

Seven practices are outlined below. They are not independent of one another: the downstream practices (verification, writer/reviewer review, agentic delegation) depend on the upstream ones (planning, task decomposition, context management) being established first.

---

## Workflow 1: The Explore-Plan-Implement Loop

**Description:** The most common failure mode in AI-assisted development is asking Claude to implement something before either party understands the problem. When Claude jumps directly to code without a planning phase, it frequently solves the wrong problem — or solves the right problem in a way that conflicts with existing architecture, requiring rework that erases the velocity gain.[^2] This is not an AI capability limitation; it is a workflow gap.

Anthropic's recommended four-phase pattern — Explore, Plan, Implement, Commit — addresses this directly. In the Explore phase, the engineer uses Plan Mode to have Claude read relevant files and understand the codebase without making any changes. In the Plan phase, Claude produces a detailed implementation plan that the engineer can edit before any code is written. Only after the plan is approved does implementation begin. This separation is not overhead — it is the mechanism by which a single AI session produces correct first-pass results rather than multiple cycles of correction.[^3]

For a small team where review bandwidth is limited, the loop has a direct practical benefit: it reduces the volume of downstream correction cycles. Engineers working with this pattern report fewer back-and-forth revisions than those who prompt directly for code.[^4]

Not every task benefits from this structure. For changes that can be described in a single sentence — a variable rename, a log line, a bug with a clear fix — the planning step adds friction without benefit. The loop is most valuable when the change touches multiple files, when the engineer is unfamiliar with the affected code, or when the correct implementation approach is not already obvious.[^2]

**Recommended Practice:**
- Default to Plan Mode for any change that touches more than two files or involves unfamiliar code; skip it for changes where the correct diff is already clear.[^2]
- Before giving Claude an implementation task, require that it reads the relevant files in Explore mode first — without making changes. "Read X and understand how Y works" should precede "implement Z."[^3]
- Have engineers edit the generated plan directly before approving implementation. A plan that requires revision is still faster than debugging incorrect code post-implementation.[^4]
- The architect should adopt the pattern explicitly and model it in team-visible sessions; the team's approach tends to converge toward whatever senior engineers visibly do.[^5]

---

## Workflow 2: Task Decomposition and Selection

**Description:** Claude Code performs significantly better on bounded, well-specified tasks than on open-ended, large-scope ones. The temptation — particularly under velocity pressure — is to hand an entire feature brief to Claude and accept whatever comes back. This tends to produce code that is locally coherent but architecturally inconsistent, and may include hallucinated utility functions, outdated API references, or patterns that conflict with the existing stack.[^6]

Effective task decomposition is a distinct skill from traditional sprint planning. The relevant unit is not a user story but a prompt: a single, verifiable task Claude can complete in one session without exceeding its context window. Phillip Carter at Honeycomb describes the principle directly: "Don't generate an entire website at once. Generate a single component, then another, then another." Each increment produces something testable, and test failure feeds back into the same session as a correction signal.[^7]

Task selection — determining what AI should do at all — is equally important. AI tools perform best when the task is well-represented in training data (common frameworks and patterns), has a fast feedback loop (frontend components, unit tests, API endpoints), and does not require organizational context the AI cannot have (product tradeoffs, customer-specific constraints). Tasks involving novel design decisions, ambiguous requirements, or security-critical logic benefit from human-first drafting, with AI assisting in refinement rather than leading.[^7]

**Recommended Practice:**
- Decompose features into AI-sized units before beginning a session: individual functions, single endpoints, isolated components.[^6]
- Maintain a `spec.md` or task breakdown for any non-trivial feature — not as bureaucracy, but as the input document for structured AI sessions. Addy Osmani describes this as doing a "waterfall in 15 minutes": rapid structured planning that prevents wasted cycles later.[^1]
- Reserve AI-first drafting for tasks that are implementation-heavy but architecturally clear. Assign human-first drafting to tasks involving novel design decisions, ambiguous requirements, or security-critical paths.[^7]
- Establish a team norm: if a task cannot be described to Claude in one sentence, it needs to be broken down further before the session begins.[^2]

---

## Workflow 3: Context Engineering

**Description:** Every Claude Code session starts from scratch with no memory of prior sessions. The quality of output is directly proportional to the quality of the context provided — and on a team of 11, inconsistent context across sessions is a primary source of the architectural inconsistency in AI-generated code described in Issue 3.[^8]

Context engineering has become the primary technical discipline around AI-assisted development. A 2026 analysis of the shift from pair programming to agentic workflows states it plainly: "The real skill in working with coding agents is no longer prompt design — it's context engineering."[^6] On a small team, this means deliberately building and maintaining the artifacts that anchor every AI session to the team's decisions: the `CLAUDE.md` file, architecture notes, naming conventions, and a shared spec format.

Boris Cherny, who created Claude Code, has documented his team's practice directly: the `CLAUDE.md` is updated multiple times per week. Any time Claude does something incorrectly, the correction is added to the file so it does not happen again.[^5] This creates a compounding asset — the file becomes incrementally more accurate over time, reducing correction frequency and cost across the whole team.

CLAUDE.md has a critical failure mode: when it grows too long, Claude begins ignoring portions of it because important rules are buried in noise. Anthropic's guidance is direct: if Claude already does something correctly without an instruction, delete it.[^2] The file should contain only things whose removal would cause Claude to make mistakes.

**Recommended Practice:**
- Maintain a single team-owned `CLAUDE.md` checked into git — not optional, but a required session artifact consulted at the start of every non-trivial session.[^5]
- Assign the architect ownership of this file, with an expectation that it is updated whenever a significant design decision is made or a recurring AI error is identified.[^5]
- Keep it short enough to be fully read: test it by asking whether Claude's behavior shifts when a rule is added or removed. Bloated files become inert.[^2]
- Supplement with a feature-specific `spec.md` for non-trivial work — requirements, edge cases, and architecture context that the global CLAUDE.md does not cover.[^1]
- Use CLAUDE.md imports to avoid duplication: reference separate files for git workflow, API conventions, and testing requirements rather than copying content inline.[^2]

---

## Workflow 4: Session Hygiene

**Description:** Claude's context window fills fast. A single debugging session or broad codebase exploration can consume tens of thousands of tokens, and as the window fills, performance degrades — Claude begins to "forget" earlier instructions, make more mistakes, and produce outputs inconsistent with earlier decisions in the same session.[^2] On a team where sessions span multiple concerns and extend across long stretches of work, context management directly determines output quality.

The two most common failure modes are "the kitchen sink session" — one session spanning multiple unrelated tasks, filling context with irrelevant material — and "the correction spiral" — Claude making the same mistake repeatedly because failed attempts are accumulating in context rather than being cleared.[^2] Both have simple remedies, but they require engineers to recognize the pattern and act before the session fully degrades.

Parallel sessions — running multiple Claude instances simultaneously on separate git worktrees — represent a different approach to scale. Boris Cherny described running five instances simultaneously as his primary productivity strategy, using system notifications to coordinate across them, calling it "the single biggest productivity unlock."[^5] For a team our size, even two or three parallel sessions on isolated branches can parallelize work that would otherwise be sequential.

**Recommended Practice:**
- Run `/clear` between unrelated tasks to reset context entirely. If a session has required more than two corrections on the same issue, clear and start with a better-specified prompt rather than continuing to correct.[^2]
- Use `/compact` with explicit scope instructions when long sessions must continue: `/compact Focus on the API changes.` This preserves relevant context while discarding noise.[^2]
- Use `--continue` or `--resume` when returning to a task across sessions, rather than re-explaining context from scratch.[^2]
- For investigation tasks, use subagents: "Use a subagent to investigate how our authentication handles token refresh." The subagent reads files and reports back a summary without consuming the main session's context window.[^2]
- Name sessions descriptively and use separate worktrees for parallel workstreams. Two focused sessions often outperform one long unfocused one.[^5]
- Commit after each successful task completion — treating commits as save points that allow safe session resets without losing progress.[^1]

---

## Workflow 5: Verification-Driven Development

**Description:** Claude performs dramatically better when it can verify its own work. Without verification criteria — a test suite, a linter, a screenshot comparison, a build check — Claude produces outputs that look plausible but may not function correctly, and the engineer becomes the only feedback loop. Every mistake requires human attention and a manual correction cycle.[^2]

This is not a theoretical principle. Anthropic's documentation identifies providing verification as "the single highest-leverage thing you can do" in a Claude Code session.[^2] Boris Cherny cites it as his number one most important tip: "Give Claude a way to verify its work. If Claude has that feedback loop, it will 2–3x the quality of the final result."[^5] Engineers who integrate test suites into their sessions — where Claude runs tests, sees failures, and iterates until they pass — consistently produce better output with less post-merge debugging than those who review AI output manually.[^3]

On our team, the risk is that verification infrastructure is uneven: frontend and backend coverage varies, and QA involvement in AI-generated code is not yet standardized. Where tests don't exist, adding even simple integration or smoke tests before a significant AI-assisted development session pays compounding returns. The more verification Claude has available, the less human review is required to catch the same class of issues.[^3]

**Recommended Practice:**
- Require verification criteria in every AI task: a test to run, an output to compare, a build to pass. "Implement X and run the test suite" produces materially better results than "implement X" alone.[^2]
- For UI changes, use browser-based verification: have Claude take a screenshot of its output and compare it against the design before the engineer reviews it.[^2]
- Configure CI/CD hooks to run automatically at session completion, catching issues before they reach PR review.[^3]
- Before beginning a significant AI-assisted feature, invest in establishing the test infrastructure the AI can verify against — tests written upfront anchor the implementation in a way that specification text alone does not.[^7]
- Use a secondary Claude session as a post-implementation review pass: a fresh-context session reviewing completed code will surface issues the implementing session rationalized away.[^5]

---

## Workflow 6: The Writer/Reviewer Pattern

**Description:** AI-generated code reviewed by the same session that wrote it has a structural blind spot: the model tends to rationalize its own decisions and downweight concerns it already implicitly addressed during writing. This is analogous to the human problem of reviewing your own work — the reviewer's insider knowledge makes them a worse reviewer, not a better one. A session that wrote code inherits assumptions from that writing process that bias its review of that same code.[^2]

The writer/reviewer pattern addresses this by separating implementation from review into two independent Claude sessions with separate contexts. One session implements; a second session — with no knowledge of the first — reviews the output cold. Because the second session has no commitment to the implementation choices, it is more likely to catch edge cases, inconsistencies with existing patterns, and architectural concerns that the implementing session's context would have obscured.[^9]

This pattern maps naturally onto human review responsibilities: the implementing engineer operates the writer session, and the architect or a second engineer operates the reviewer session. The fresh-context review is not a replacement for human code review but a preprocessing step that makes human review faster — focused on genuine design concerns rather than catching the same AI-specific errors a reviewer session would catch automatically.[^4]

**Recommended Practice:**
- For any non-trivial AI-generated PR, run a separate reviewer session before human review: "Review @src/path/to/feature for edge cases, race conditions, and consistency with our existing patterns." Share the output with the human reviewer.[^2]
- Ensure the reviewer session has access to the `CLAUDE.md` and architectural context — a reviewer that does not know the team's conventions cannot catch violations of them.[^5]
- Assign the architect as the default operator of the reviewer session for security-critical or architecturally significant code.[^9]
- Maintain the separation clearly: the writer session should not be shown the reviewer's feedback and asked to defend its choices. If revisions are needed, start a fresh writer session. This is what preserves the independence of the review.[^2]

---

## Workflow 7: Agentic Delegation

**Description:** Claude Code supports fully autonomous, long-running workflows where an engineer delegates a task, steps away, and returns to a completed result. Non-interactive mode (`claude -p`), fan-out across files, and agent teams represent a qualitative shift from AI as a pair programmer to AI as an autonomous worker. As the CIO framing puts it, we are moving through three stages: assistance (AI supports discrete tasks), augmentation (AI manages multi-step workflows), and autonomy (AI operates across domains guided by objectives).[^9]

The risk of premature delegation is significant. Anthropic's 2026 Agentic Coding Trends Report found that developers can fully delegate only 0–20% of tasks, with the remaining 80–100% requiring "constant collaboration requiring supervision, validation, and human judgment."[^8] Delegating tasks outside that 20% — tasks with ambiguous requirements, novel design decisions, or security implications — produces AI-generated code that is architecturally wrong in ways that take longer to debug than to write manually.[^7]

The engineering judgment required is knowing which tasks fall within the delegable 20%. Well-delegable tasks share identifiable characteristics: requirements are fully specified, affected files are narrow and well-understood, the task pattern has high precedent in the codebase (e.g., a new endpoint that mirrors ten existing ones), verification criteria exist, and the output will be reviewed before merge.[^8] Tasks involving cross-cutting architectural decisions, unknown territory, or production-sensitive paths require interactive oversight regardless of automation capability.

**Recommended Practice:**
- Reserve agentic delegation for tasks that meet all delegability criteria: fully specified requirements, narrow scope, high-precedent patterns, existing verification, and a review step before merge.[^8]
- Use the fan-out pattern for large-scale migrations: generate a task list, write a loop script using `claude -p`, test on two or three files before running at scale, and use `--allowedTools` to scope permissions for unattended runs.[^2]
- For complex multi-agent workflows, define a clear orchestrator role and bounded agent responsibilities. Agents working without defined scope tend to wander into inconsistent or uncompilable output.[^6]
- Establish a team norm: any task delegated in auto mode must have a corresponding human review step before merge. Delegation reduces effort, not accountability.[^8]
- The architect and team lead should audit delegation patterns quarterly: which task types are being delegated, what the error rate looks like, and whether scope has crept beyond the delegable 20%.[^5]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Explore-Plan-Implement Loop | Default to Plan Mode for multi-file changes | All engineers |
| Task Decomposition | Require spec.md for non-trivial features | Architect |
| Context Engineering | Team CLAUDE.md checked into git, architect-owned | Architect |
| Session Hygiene | /clear between unrelated tasks; named worktrees for parallel work | All engineers |
| Verification-Driven Development | Require verification criteria in every AI task | All engineers |
| Writer/Reviewer Pattern | Separate reviewer session before human PR review | Architect + engineers |
| Agentic Delegation | Delegate only fully-specified, narrow-scope, high-precedent tasks | Engineering team |

---

[^1]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    - "Waterfall in 15 minutes": the argument for rapid upfront spec.md planning before any code generation begins, preventing wasted cycles in subsequent sessions
    - Task decomposition: why monolithic prompts produce jumbled, architecturally inconsistent output and how iterative chunking prevents it
    - Commit discipline: treating frequent granular commits as save points for safe session resets and experiment rollbacks

[^2]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    - Explore-Plan-Implement loop: the four-phase workflow, when to use it vs. skip it, and the context overhead tradeoff
    - CLAUDE.md discipline: why over-specified context files cause Claude to ignore instructions, and the pruning test
    - Session hygiene: /clear, /compact, subagent delegation, common failure patterns (kitchen sink session, correction spiral)
    - Verification: identified as "the single highest-leverage thing you can do" in a Claude Code session

[^3]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    - Practical verification setup: integrating test suites, CI hooks, and screenshot comparison into the AI session loop
    - Plan Mode specifics: entering it, what it prevents, and example prompts for each phase of the Explore-Plan-Implement cycle

[^4]: Anthropic — "Eight Trends Defining How Software Gets Built in 2026," Anthropic, 2026. https://claude.com/blog/eight-trends-defining-how-software-gets-built-in-2026
    - Human delegation ceiling: developers use AI in approximately 60% of their work but can "fully delegate" only 0–20% of tasks, with the rest requiring active collaboration
    - Workflow shift: engineers moving from writing code to coordinating agents, reviewing outputs, and owning architectural decisions

[^5]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    - Parallel sessions: running 5 instances simultaneously on separate git worktrees as the "single biggest productivity unlock"
    - CLAUDE.md discipline: updating the file multiple times weekly; using PR comments tagged @.claude to feed corrections back into the shared context file
    - Verification as primary tip: "Give Claude a way to verify its work. If Claude has that feedback loop, it will 2–3x the quality of the final result."

[^6]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    - Context engineering as successor to prompt engineering: "The real skill in working with coding agents is no longer prompt design — it's context engineering"
    - Agentic execution loops: the emergence of long-running autonomous workflows and how they change the fundamental interaction pattern between developer and AI

[^7]: Phillip Carter — "How I Code With LLMs These Days," Honeycomb, March 2025. https://www.honeycomb.io/blog/how-i-code-with-llms-these-days
    - Task suitability framework: three factors (task commonality, code availability, feedback loop speed) that determine AI effectiveness by task type
    - Incremental generation principle: generating small testable pieces rather than entire systems
    - Agent guardrail warning: agents currently struggle with existing codebases and non-trivial tasks without firm scope constraints and human oversight

[^8]: Anthropic — "2026 Agentic Coding Trends Report," Anthropic, 2026. https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf
    - Delegation ceiling: developers can fully delegate only 0–20% of tasks; 80–100% require active supervision, validation, and human judgment
    - Real-world benchmarks: Rakuten completed a vLLM implementation autonomously in 7 hours with 99.9% numerical accuracy; TELUS achieved 30% faster engineering code shipping across 500,000+ hours saved

[^9]: CIO — "How Agentic AI Will Reshape Engineering Workflows in 2026," April 2026. https://www.cio.com/article/4134741/how-agentic-ai-will-reshape-engineering-workflows-in-2026.html
    - "Delegate, Review, Own" operating model: AI handles first-pass execution; engineers review for correctness and risk; humans retain ownership of architecture and outcomes
    - Three-stage evolution framework: assistance → augmentation → autonomy, and the workflow implications of each transition

[^10]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
    - Plan-first workflow: how iterating on a detailed plan before switching to implementation mode produces "1-shot implementations" that require minimal correction
    - Parallel session coordination: workflow for managing multiple simultaneous Claude instances with system notifications and named sessions
    - Opus model rationale: why using the heaviest, slowest model often produces faster net results due to reduced steering and better tool use
