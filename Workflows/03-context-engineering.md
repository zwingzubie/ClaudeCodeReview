## Overview

Context engineering is the discipline of deliberately building and maintaining the information structures that govern every AI coding session. It has emerged as the primary technical skill in AI-assisted development — superseding prompt engineering as the lever that separates teams getting consistent, architecturally coherent output from teams getting inconsistent results that require extensive correction.[^1]

Every Claude Code session starts from scratch. The quality of its output is bounded by the quality of the context it receives. On a team of 11, each engineer runs multiple sessions per day; without shared, well-maintained context infrastructure, each session operates on a different model of the codebase. The aggregate effect is the architectural drift described in Issue 3 — not from malice or carelessness, but from the absence of a context layer that enforces shared conventions.

This memo covers the four layers of context engineering: the `CLAUDE.md` shared team document, the `spec.md` feature-level specification, session-level context management, and multi-agent context coordination.

---

## Section 1: The CLAUDE.md Architecture

**Description:** `CLAUDE.md` is the file Claude reads at the start of every session. It is the team's persistent context layer — the architectural contract that ensures every engineer's AI sessions operate with the same conventions, constraints, and understanding of the codebase. For a team our size, it is the single highest-leverage context artifact available, and the architect's most important maintainability tool.[^2]

Its value compounds over time. Boris Cherny, who created Claude Code, has described his team's practice: `CLAUDE.md` is updated multiple times per week. Any time Claude does something incorrectly, the correction is added to the file so it does not recur.[^3] Over months, this produces an increasingly accurate behavioral contract — one that eliminates whole categories of AI error that would otherwise require individual correction in every session.

`CLAUDE.md` has a known failure mode: over-specification. When the file grows too long, Claude begins ignoring portions of it because important rules are buried in noise. Anthropic's documented guidance is precise: if Claude already does something correctly without an instruction, delete it. If a line's removal would not cause Claude to make mistakes, cut it. The file is not a documentation archive; it is a behavioral controller.[^2]

The file supports imports using `@path/to/file` syntax, enabling modular organization: a root `CLAUDE.md` that imports separate files for git workflow, API conventions, testing requirements, and security rules. This allows the architect to own the root file while delegating module-specific context to the engineers who own those modules.[^2]

**Recommended Practice:**
- Run `/init` on the project to generate a starter `CLAUDE.md` from the current codebase structure, then refine rather than writing from scratch.[^2]
- Keep the file short enough to be fully read: apply the "would removing this cause a mistake?" test to every line. Prune aggressively; err on the side of shorter.[^2]
- Check the file into git. Update it after every significant architectural decision or recurring AI error. Tag PRs with `@.claude` to route corrections back to the file as a standard review step.[^3]
- Assign the architect as the owner with merge authority. Any engineer can propose additions via PR; the architect approves and prunes.[^4]
- Structure the file with imports for domain-specific context: `CLAUDE.md` imports `docs/api-conventions.md`, `docs/testing-standards.md`, and `docs/security-rules.md`. This scales as the codebase grows without bloating the root file.[^2]

---

## Section 2: Spec-Driven Development

**Description:** Spec-driven development (SDD) is a workflow where specifications — not code — are the primary artifact of software development. Code becomes an output derived from human-authored specifications, rather than the primary deliverable. The specification captures requirements, architecture decisions, edge cases, and acceptance criteria before any implementation begins.[^5]

The workflow has four phases. In the Requirements phase, the specification captures what is being built and why — user-visible behavior, not implementation details. In the Design phase, it captures how it will work: data models, API contracts, module responsibilities, dependency relationships. In the Tasks phase, requirements and design are decomposed into specific, ordered implementation steps. Only in the Implementation phase does any code get written — by AI, executing against the spec.[^5]

This is not a heavyweight process. Addy Osmani describes creating a full-featured specification as doing "a waterfall in 15 minutes."[^6] For a 2-3 day feature, a 15-minute spec session that prevents a day of rework is a strongly positive trade. The spec also serves a second function: it is the context document for AI sessions, eliminating the need to reconstruct requirements from memory at the start of each session.

The spec-driven workflow has demonstrated measurable leverage: one engineer completed a SQLite-to-IndexedDB migration — a 2-3 day manual estimate — in one afternoon by breaking the spec into 14 discrete implementation tasks and delegating each to a subagent. Main session context usage remained at 71% (143k of 200k tokens) despite coordinating multiple parallel subagents.[^7]

**Recommended Practice:**
- Write a `spec.md` for any feature estimated at more than half a day of work. The spec should cover requirements, design decisions, task breakdown, and acceptance criteria.[^5]
- Use Claude to draft the spec from a brief, then edit the draft rather than writing from scratch. The interview pattern — "Ask me questions using AskUserQuestion until you have everything you need to write a spec" — surfaces ambiguities that would otherwise appear mid-implementation.[^8]
- Store specs in a `/docs/specs/` directory alongside the code they describe. They are living documents: update them when design changes, and note what changed and why.[^4]
- After implementation, conduct a brief spec-vs-implementation comparison: does the code match the spec? Divergences should either update the spec or the code, not be silently ignored.[^5]

---

## Section 3: Session-Level Context Management

**Description:** Beyond project-level context (`CLAUDE.md`) and feature-level context (`spec.md`), individual AI sessions require deliberate context management to avoid the degradation that emerges as context windows fill. Claude's context window holds every message, file read, and command output from the session. As it fills, performance degrades — Claude may start ignoring earlier instructions or making errors that would not occur in a fresh session.[^2]

The key session-level context techniques are: scoped investigation (asking Claude to read specific files rather than broad codebase exploration), subagent delegation for research tasks (investigation runs in a separate context that reports a summary rather than consuming the main session's window), and the `/compact` command for long sessions that must continue.[^2]

Phillip Carter at Honeycomb identified an underused technique: context-self-updating. When a code component is verified to work correctly, ask Claude to update the session's context documentation to reflect it. Over the course of a long session, this creates an evolving knowledge base that reduces correction frequency.[^9]

Context window usage can be tracked with a custom status line that displays remaining tokens. This gives engineers a concrete signal to act — run `/compact`, start a fresh session, or switch to a subagent — rather than discovering degradation only after output quality has already dropped.[^3]

**Recommended Practice:**
- Use subagents for investigation tasks: "Use a subagent to investigate how the authentication module handles token refresh, and report a summary." This preserves the main session's context for implementation.[^2]
- Run `/compact Focus on [specific area]` in long sessions rather than starting fresh if implementation state is significant. Use explicit scope instructions to preserve what matters.[^2]
- Configure a context window status line to display real-time token usage. Act on this signal proactively rather than reactively.[^3]
- After verifying a component works, instruct Claude to update its understanding: "Update your notes on how session management works now that we've verified this implementation." This reduces re-explanation in later turns.[^9]
- Never ask Claude to "look through the codebase" without a specific scope. Unbounded exploration is the fastest path to context window exhaustion.[^2]

---

## Section 4: Multi-Agent Context Coordination

**Description:** As agentic workflows mature, context engineering extends from single sessions to multi-agent coordination — ensuring that multiple Claude instances working in parallel share the same architectural understanding and do not make conflicting decisions.[^7]

The primary mechanism for multi-agent context coordination is the shared `CLAUDE.md` combined with subagent definitions in `.claude/agents/`. Each subagent definition includes a system prompt that establishes its role, constraints, and the architectural context it should maintain. A security reviewer subagent and a performance analyzer subagent, for example, share the same architectural context from `CLAUDE.md` but have different expertise focus and tool access.[^2]

For parallel subagent workflows, the orchestrator session coordinates context sharing by assigning specific files and modules to each subagent, setting explicit boundaries that prevent overlapping file edits, and requiring each subagent to report its changes before the orchestrator commits. This is the pattern used in the spec-driven 14-task migration described above: each subagent received a bounded task with specific files in scope, preventing context collisions.[^7]

The failure mode in multi-agent context coordination is conflicting assumptions: two agents making incompatible decisions about shared state, interfaces, or data formats because neither was given explicit context about what the other would do. Mitigation requires explicit coordination instructions in the orchestrator's plan: "Subagent A handles the data model; Subagent B handles the API layer; Subagent B must wait for Subagent A's type definitions before beginning."[^7]

**Recommended Practice:**
- Define subagent personas in `.claude/agents/` for recurring specialized tasks: security reviewer, performance analyzer, test generator. Each should include relevant architectural context from `CLAUDE.md` in its system prompt.[^2]
- For parallel subagent workflows, establish explicit dependency ordering before delegation: which subagents must complete before others can begin? This prevents conflicting implementations.[^7]
- Require each subagent to report what it changed before the orchestrator session commits. The orchestrator should perform a coherence check before merging parallel work.[^7]
- Store subagent definitions in git alongside `CLAUDE.md`. Treat them as team infrastructure with the same review and update process.[^2]

---

## Section 5: Context Degradation and Refresh

**Description:** Context quality degrades in predictable patterns over the lifecycle of a feature. Early sessions have fresh context and high output quality. Mid-feature sessions accumulate implementation history, correction cycles, and exploratory dead ends that pollute the context with noise. Late sessions operating in context-compacted state risk losing architectural decisions made early in the feature's development.

The `CLAUDE.md` and `spec.md` together address this by externalizing the most important context into persistent files that survive session resets. When a long session degrades, the correct action is often to start a fresh session — pointing Claude to the spec.md and CLAUDE.md for context — rather than continuing in a polluted session. This requires the spec.md to be current: a spec that diverged from the implementation weeks ago provides incorrect context in the new session.[^5]

The GitHub blog's 2026 work on spec-driven development describes maintaining a "living specification" that is updated throughout implementation, not just at the beginning. This living specification approach keeps the feature-level context fresh regardless of session resets, and becomes the primary artifact for team review and onboarding after the feature ships.[^10]

**Recommended Practice:**
- Treat `spec.md` as a living document: update it as design decisions evolve during implementation. A spec that diverges from code is worse than no spec.[^5]
- When a mid-feature session shows quality degradation, start fresh. Point the new session to `CLAUDE.md` + `spec.md` + specific relevant files. This reset is faster than attempting to repair a polluted session.[^2]
- After a feature ships, archive the spec alongside the PR as documentation of design intent. This is more valuable for future maintainers than inline comments because it captures the "why" rather than the "what."[^10]
- Run a quarterly `CLAUDE.md` review with the architect. Remove outdated rules, add new ones from recurring corrections, and restructure as the codebase evolves.[^3]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| CLAUDE.md Architecture | Run /init, refine, check into git; assign architect as owner | Architect |
| Spec-Driven Development | Write spec.md before any feature > half a day; use interview pattern for ambiguity | All engineers |
| Session Context Management | Configure context window status line; use subagents for investigation | All engineers |
| Multi-Agent Coordination | Define subagent personas in .claude/agents/; explicit dependency ordering | Architect |
| Context Refresh | Treat spec.md as living document; start fresh sessions over polluted ones | All engineers |

---

[^1]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    - Context engineering as the successor to prompt engineering: "the real skill is no longer prompt design — it's context engineering"
    - Shift from interactive to spec-driven context: developers writing detailed feature specifications that agents reference throughout implementation
    - Long-running autonomous workflows and the role of persistent context in enabling them

[^2]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    - CLAUDE.md architecture: imports syntax, pruning rules, the "would removing this cause a mistake?" test
    - Session context management: /clear, /compact with scope instructions, subagent delegation for investigation
    - Subagent definitions in .claude/agents/ for specialized recurring tasks

[^3]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    - CLAUDE.md discipline: update multiple times weekly; @.claude PR tagging to route corrections back to the file
    - Context window status line configuration for real-time token usage tracking
    - Plan Mode as context control: iterating on plans before auto-accept to preserve context for implementation

[^4]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    - Three-tier specification hierarchy: product-level, change-level, implementation plans
    - Style guides and architectural context as inputs to AI sessions, not afterthoughts
    - "Big Daddy" anti-hallucination rule: explicit no-fabrication instructions in context files

[^5]: Heeki Park — "Using Spec-Driven Development with Claude Code," Medium, March 2026. https://heeki.medium.com/using-spec-driven-development-with-claude-code-4a1ebe5d9f29
    - Spec-driven development workflow: four phases from requirements through implementation
    - Living specification discipline: updating specs when design changes, not just at feature start
    - Context window management in sustained development: Sonnet 4.6 sustaining several hours vs. Opus 4.6 hitting limits in 45 minutes

[^6]: Addy Osmani at Google / HAMY — "5 AI Coding Best Practices from a Google AI Director," HAMY.xyz, January 2026. https://hamy.xyz/blog/2026-01_ai-engineering-best-practices
    - "Waterfall in 15 minutes": rapid structured planning that prevents wasted implementation cycles
    - CLAUDE.md and equivalent agent files as the codebase constitution
    - Comprehensive context requirement: codebase structure, product requirements, patterns, verification steps

[^7]: Artur Less — "Spec-Driven Development with Claude Code," Level Up Coding, March 2026. https://levelup.gitconnected.com/spec-driven-development-with-claude-code-1b08184965e3
    - Multi-agent context coordination: 14-task delegation with explicit dependency ordering
    - 71% context usage (143k/200k tokens) despite coordinating multiple parallel subagents
    - Orchestrator role clarity: "you are the main agent and your subagents are your devs"

[^8]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    - Interview pattern: "let Claude interview you using AskUserQuestion" to surface spec ambiguities
    - Subagent delegation mechanics and context isolation
    - Worktree integration for parallel sessions with context isolation

[^9]: Phillip Carter — "How I Code With LLMs These Days," Honeycomb, March 2025. https://www.honeycomb.io/blog/how-i-code-with-llms-these-days
    - Context-self-updating technique: asking Claude to update its own understanding when a component is verified
    - RAG approach to context building: using llms.txt standard and project rules files
    - Incremental context accumulation over a feature's development lifecycle

[^10]: GitHub Blog — "Spec-Driven Development with AI: Get Started with a New Open Source Toolkit," GitHub, 2026. https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/
    - Living specification approach: specs as primary artifacts that evolve throughout implementation
    - Post-ship archival: specs as design documentation capturing "why" rather than "what"
    - Open source toolkit for spec-driven workflows across AI coding platforms

[^11]: Brian Chambers — "Chamber of Tech Secrets #54: Spec-Driven Development," Substack, 2026. https://brianchambers.substack.com/p/chamber-of-tech-secrets-54-spec-driven
    - Requirements → Design → Tasks → Implementation as the canonical SDD workflow
    - CLAUDE.md as the "steering document" that governs all sessions alongside SKILL.md files
    - Practitioners reporting that upfront specification investment reduces total development time even for small features
