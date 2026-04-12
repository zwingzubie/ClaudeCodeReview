## Context Optimization

**Related to:** [Cost & Token Economics Overview](00-overview.md) — Cost Area 3 · [Workflows: Session Hygiene](../Workflows/04-session-hygiene.md) · [Workflows: Context Engineering](../Workflows/03-context-engineering.md) · [Tooling: CLAUDE.md Configuration](../Tooling & Configuration/01-claude-md-configuration.md)

---

## Overview

Context optimization is the discipline of sending the right content — not the maximum content — into a Claude Code session. It is the most consistently underused cost lever available to engineering teams because the failure mode it addresses is invisible until the invoice arrives: context inflation accumulates silently turn by turn, the model continues to respond, and nothing in the session UI signals that the per-turn cost has tripled since the session began. Teams that understand context accumulation and manage it proactively spend 40–60% less per session than teams running equivalent tasks with unmanaged context.[^1]

The core principle is that more context is not unconditionally better. Relevant, high-density context improves output quality; irrelevant context dilutes the model's attention without improving quality, and costs the same per token as relevant context. An engineer who injects an entire module directory to answer a question about one file is paying for the content of 15 files to help with 1. An engineer who allows conversation history to accumulate across 30 turns without compacting is paying for early exploration turns that are no longer relevant to the current implementation task. Context optimization is about ensuring that every token in the context window is earning its cost.[^2]

---

## Section 1: Scoping Sessions Before They Start

**Description:** The highest-leverage context optimization intervention is pre-session scoping: defining what the session will address, what files are in scope, and what the session will not attempt before the first message is sent. A session with explicit scope boundaries accumulates less irrelevant context than an open-ended session because the scope definition constrains what the model considers relevant to inject, what the engineer is tempted to ask tangentially, and how far the session's questions wander from its original purpose.[^3]

Pre-session scoping also produces better output, independent of cost. A session that knows its task boundary can optimize its exploration toward that boundary rather than surveying broadly. The quality and cost improvements from scoping are correlated: the same discipline that prevents cost overrun also prevents the output dilution that comes from a session trying to address more than it can address well.[^2]

**Recommended Practice:**
- Before starting any non-trivial session, write a one-paragraph session brief: what specific outcome is expected, which files are directly relevant, which files may need to be referenced, and what the session will not address. This brief can be the first message of the session, creating both a session record and a scope boundary visible to the model.[^3]
- Use the spec.md workflow (see Templates: spec.md) for feature implementation sessions. A well-formed spec document pre-loads the relevant requirements, constraints, and acceptance criteria at session start, eliminating the need for exploratory questions that accumulate context without advancing the task.[^4]
- Define a termination criterion for every agentic session: "This session completes when [specific outcome] is verified." Agentic sessions without termination criteria tend to extend past the point of diminishing returns because there is no forcing function to stop. A defined termination criterion also provides a cost check: if the session has not reached the criterion in N turns, evaluate whether to continue or reset.[^1]
- Resist the temptation to expand session scope mid-session. When a session surfaces an adjacent problem — a related bug, a possible improvement, an interesting refactor — record it for a future session rather than addressing it in the current one. Scope expansion mid-session is the most common source of unplanned context growth.

---

## Section 2: Targeted File Injection

**Description:** Claude Code can read files and directories as context injection. The cost and quality implications of injection scope differ significantly between targeted file injection (specific files relevant to the task) and directory-level injection (all files in a module or directory). For a module with 15 files averaging 300 lines each, directory injection adds approximately 45,000 tokens of context that may be largely irrelevant to a task touching one or two of those files. Targeted injection of the two relevant files adds approximately 6,000 tokens — a 7.5× context reduction for the same output quality on a focused task.[^2]

The quality case for targeted injection is as important as the cost case. Irrelevant context does not just cost money — it competes for model attention with relevant context. On tasks where the correct answer depends on subtle details in specific files, diluting the context with unrelated code can reduce the model's effective attention to those details. Targeted injection is therefore both cheaper and, for focused tasks, often higher quality than broad injection.[^5]

**Recommended Practice:**
- Before injecting files, ask: which files does this session actually need to read to complete its task? Inject those files specifically rather than their containing directory. This requires 30 seconds of pre-session thought that can save 20,000–80,000 tokens of unnecessary context per session.[^2]
- For refactoring sessions that span multiple files, inject files in phases: inject the files needed for the first phase, complete that phase, compact, then inject the files needed for the second phase. This prevents the full refactoring context from accumulating in a single session window.[^1]
- Use CLAUDE.md to document which files in the codebase are structural references — files that should almost always be included in sessions touching a given module (e.g., the module's primary interface file, its configuration schema, its test helper). These are the files worth including by default; all other files should be added only when specifically needed.[^3]
- When a session needs to understand a pattern from an existing file without modifying it, consider asking Claude to read a relevant excerpt rather than the whole file: "Read lines 45–120 of auth_middleware.ts, which contains the token validation logic." Partial reads cost proportionally less than full file reads and are sufficient when the full file is not needed.[^5]

---

## Section 3: Conversation History Management

**Description:** Conversation history is the largest source of context growth in active sessions. Each turn adds to the running total of tokens that must be sent with every subsequent request — including the model's own responses, which are typically the most verbose part of the history. An unchecked session where the model produces 800-token responses over 20 turns has accumulated 16,000 tokens of model output in history alone, in addition to the growing human input history. By turn 20, this history may constitute 60–70% of the total input context.[^1]

The `/compact` command addresses this directly by replacing the conversation history with a summarized version, preserving the relevant context for the current task phase while discarding detail from earlier phases that is no longer relevant. Used at phase boundaries — after exploration and before implementation, after implementation and before testing — `/compact` resets the cost curve without losing task continuity. The cost reduction is measurable: a session compacted at turn 10 before continuing to turn 20 costs roughly half as much for turns 11–20 as the same session without compaction.[^6]

**Recommended Practice:**
- Compact at task phase transitions as a discipline, not just when context feels unwieldy. The optimal time to compact is when a coherent phase of work is complete and a different phase is beginning — this timing produces the most informative compact summary while discarding the most irrelevant detail.[^6]
- When compacting, review the compact summary Claude produces before proceeding. The summary should capture the key decisions, constraints, and outcomes from the compacted history. If the summary omits something important, add it explicitly as context for the next phase rather than relying on the compact to have captured it.[^6]
- Consider starting a new session rather than compacting when a session has drifted significantly from its original scope. A fresh session with a clear brief and the relevant output from the prior session as starting context is often cheaper and higher quality than a heavily compacted session with accumulated context baggage.[^3]
- For sessions with multiple distinct phases (explore → plan → implement → test), plan the compact points before the session begins. Knowing in advance that you will compact after exploration and after planning allows you to be more thorough in each phase without worrying about context accumulation — the compact resets the cost curve at the planned point.[^2]

---

## Section 4: CLAUDE.md Optimization for Cost

**Description:** CLAUDE.md is sent with every session — it is the highest-frequency context document in the team's AI workflow. A CLAUDE.md file that has grown to 15,000 tokens through incremental addition without structural review is costing the team 15,000 tokens of input context per session turn, regardless of whether most of that content is relevant to the session's task. The cost of CLAUDE.md is not a one-time cost — it multiplies with every session turn for the lifetime of the file. CLAUDE.md optimization is therefore a recurring maintenance task, not a one-time configuration exercise.[^4]

The structure of CLAUDE.md also affects prompt caching efficiency. Content that changes frequently near the top of the file invalidates the cache for all content below it, converting cache hits to cache misses. A CLAUDE.md that mixes stable architectural principles with frequently updated sprint notes is structuring its cache-unfriendly content in the worst possible position.[^7]

**Recommended Practice:**
- Audit CLAUDE.md quarterly for content that is no longer current, no longer useful, or that could be moved to task-specific injected files rather than the global context. A CLAUDE.md that has not been pruned in six months likely contains 20–40% content that does not improve session quality but costs tokens every turn.[^4]
- Structure CLAUDE.md from most stable to most variable: architectural principles at the top, team conventions in the middle, current sprint context or recent change notes at the bottom or in a separate file. This structure maximizes prompt cache efficiency for the stable content and minimizes the cost of cache invalidation when variable content changes.[^7]
- Break very large CLAUDE.md files into a core CLAUDE.md (architectural principles, hard constraints) plus module-specific addenda injected only in sessions touching those modules. A 20,000-token monolithic CLAUDE.md is expensive on every session; a 5,000-token core CLAUDE.md with 5,000-token module-specific addenda costs one-third as much on sessions that only need the core.[^3]
- Measure the impact of CLAUDE.md changes by tracking session token counts before and after significant edits. A 20% reduction in CLAUDE.md size should produce a measurable reduction in per-session cost that is visible in the usage dashboard within a few days.

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Pre-Session Scoping | Add session brief as first message on all non-trivial sessions | Engineering team |
| Targeted File Injection | Audit common session patterns for unnecessary directory injection | Backend lead |
| Conversation History | Set compact checkpoints at phase transitions; document timing in team norms | Engineering team |
| CLAUDE.md Optimization | Quarterly CLAUDE.md audit; restructure for stable-to-variable ordering | Architect |

---

[^1]: Anthropic — "Managing Long Sessions," Claude Code Documentation, 2026. https://code.claude.com/docs/en/managing-long-sessions
    Context accumulation in long sessions; `/compact` mechanics and cost reset effect; agentic session termination criteria; phase-boundary compaction timing.

[^2]: The Pragmatic Engineer — "AI Tooling for Software Engineers in 2026," March 2026. https://newsletter.pragmaticengineer.com/p/ai-tooling-2026
    Targeted file injection vs. directory injection cost comparison; relevant context density as a quality driver; pre-session scoping as the highest-leverage optimization intervention.

[^3]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    Session scoping and brief patterns; scope expansion as a cost risk; module-specific CLAUDE.md addenda; fresh session vs. compacted session tradeoffs.

[^4]: Anthropic — "CLAUDE.md Configuration Guide," Claude Code Documentation, 2026. https://docs.anthropic.com/en/docs/claude-code/memory
    CLAUDE.md as a persistent cost multiplier; quarterly audit cadence; content categorization for pruning decisions; module-specific vs. global context organization.

[^5]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    Targeted partial file reads; attention dilution from irrelevant context; the quality argument for focused injection on tasks with subtle correctness requirements.

[^6]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    `/compact` command usage patterns; compact summary review as a quality gate; the cost curve reset effect measured across common session types.

[^7]: Anthropic — "Prompt Caching," Anthropic API Documentation, 2026. https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
    Cache invalidation propagation in CLAUDE.md; stable-to-variable content ordering for cache efficiency; the cost of frequent cache misses on high-frequency context documents.
