## Understanding Token Costs

**Related to:** [Cost & Token Economics Overview](00-overview.md) — Cost Area 1 · [Metrics: Session and Context Efficiency](../Metrics/00-overview.md) · [Workflows: Session Hygiene](../Workflows/04-session-hygiene.md) · [Tooling: CLAUDE.md Configuration](../Tooling & Configuration/01-claude-md-configuration.md)

---

## Overview

Token costs are the fundamental unit of economic accountability for Claude Code sessions. Every character the API receives as input — including CLAUDE.md contents, injected files, conversation history, and the engineer's typed prompt — is priced as an input token. Every character the model generates in response — code, explanation, plans, and error messages — is priced as an output token. The ratio of input to output tokens, the model tier selected, and the proportion of input tokens that hit the prompt cache together determine the cost of any given session.[^1]

The counterintuitive fact that drives most team overspend is that the engineer's visible prompt is almost never the dominant cost driver. In a typical Claude Code feature implementation session with a standard CLAUDE.md, three injected context files, and twelve conversation turns, the engineer's own words may account for 3–8% of total input tokens. The other 92–97% is structural context — content that is sent every turn, that the engineer rarely thinks about, and that accumulates over the session as conversation history grows. Teams that understand this cost structure manage it; teams that do not assume that using AI less aggressively in their prompts will meaningfully reduce costs.

---

## Section 1: Input and Output Token Mechanics

**Description:** The Claude API processes text as tokens, not characters or words. Tokens are not word-aligned — a common English word is typically one token, a less common technical term may be two or three, and a short code symbol like `{` is a fraction of a token. For practical estimation, engineers should use two rules of thumb: approximately 750 English words per 1,000 tokens, and approximately 500–600 lines of code per 1,000 tokens (depending on language verbosity). These approximations are sufficient for estimating session context size without exact tokenization.[^1]

Input and output tokens are priced differently at every model tier. Output tokens are generally two to five times more expensive than input tokens, reflecting the greater computational cost of generation versus evaluation. This means that a session producing long explanatory responses costs significantly more than a session producing the same amount of code with minimal commentary. Prompt patterns that elicit verbose model responses — asking for detailed explanations, requesting reasoning walk-throughs, using "think step by step" on low-complexity tasks — add measurable output token cost without necessarily adding engineering value.[^3]

**Recommended Practice:**
- When requesting code generation, prompt for code first and explanation optionally: "Implement X. If there is anything non-obvious about this implementation, add a brief comment." This pattern avoids paying output token cost for explanations on tasks whose implementation is already clear to the engineer.[^3]
- For tasks where understanding is more important than brevity, budget for explanation output tokens explicitly. A session that produces 2,000 tokens of code and 3,000 tokens of explanation is not inefficient if the explanation substantially reduces time-to-comprehension and subsequent debugging.
- Understand the input/output token ratio for the team's most common session patterns. If feature implementation sessions are consistently producing 5,000 tokens of output per 30,000 tokens of input, that ratio is a baseline for cost estimation. A session substantially above that output ratio is generating either unusually verbose responses or longer-than-typical conversations.[^4]
- Review model pricing at each major Anthropic model update, as pricing changes with model releases. The relative cost ratios between tiers are more stable than absolute prices; team norms built on relative ratios remain valid across pricing updates.

---

## Section 2: Context Window and Conversation History Costs

**Description:** The context window is the total content the model can see in a single request. In Claude Code, the context window fills from the bottom as the conversation progresses: early turns become older history, and each new turn adds to the running total of tokens that must be sent with every subsequent request. A session at turn 15 with 8,000 tokens of base context (CLAUDE.md, injected files) and 1,000 tokens of conversation per turn is sending approximately 23,000 tokens of input on turn 15 — nearly three times the initial context cost. By turn 30, the same session sends 38,000 input tokens per turn, nearly five times the starting cost.

This cost curve is the primary argument for proactive context management. The `/compact` command replaces the conversation history with a summarized version, substantially reducing the per-turn input cost for the remainder of the session. Without `/compact`, the cost per turn increases monotonically with session length. With strategic `/compact` usage between session phases, the cost curve resets and the session's total cost can be reduced by 30–60% relative to the uncompacted equivalent.[^5]

**Recommended Practice:**
- Use `/compact` at natural session phase boundaries: after exploration is complete and before implementation begins, after a bug is diagnosed and before a fix is written, after a refactoring pass and before testing begins. These transitions are the points where early conversation history is least relevant to the next phase.[^5]
- Set a soft context checkpoint: if a session exceeds 20 conversation turns without a `/compact`, pause and evaluate whether an explicit compact or a session reset is warranted. 20 turns is a heuristic, not a rule — the relevant question is whether early conversation history is still relevant to the current task.
- Avoid re-injecting already-present context. If the session has already read a file in a prior turn, the content is in conversation history — re-injecting the same file adds tokens without adding information. Use conversation history as the reference rather than re-reading files that were already read.[^5]
- For tasks requiring deep file analysis across a large codebase, prefer a focused session that reads only the relevant files rather than a general session with broad context access. A focused 10,000-token session with the right files often produces better results than a 50,000-token session with extensive but diluted context.

---

## Section 3: Prompt Caching Economics

**Description:** Prompt caching is the most economically significant optimization available to teams using Claude Code with consistent session structures. When the same content appears at the beginning of the context across multiple sessions, the API can serve it from cache at approximately 10% of the normal input token cost. For a team with a 10,000-token CLAUDE.md and a 5,000-token project context file that are sent on every session turn, prompt caching converts 15,000 tokens of full-price input into 15,000 tokens of cached input — a 90% cost reduction on that content for every cache hit.[^6]

The economics are significant at team scale. A team of 11 engineers each running two sessions per day with a 15,000-token cached prefix, at 10 turns per session, generates approximately 3,300,000 cached input tokens per month. At Sonnet input token pricing, the difference between cached and uncached for that volume is substantial. Prompt caching is not a marginal optimization — for teams with stable session structures, it is a fundamental cost control that should be enabled before any other optimization is addressed.[^6]

**Recommended Practice:**
- Enable prompt caching for all content that is stable across sessions: CLAUDE.md, project spec files, architecture documents, and any boilerplate context injected at session start. These are the highest-value targets for caching because they appear at the beginning of every session and are rarely modified.[^6]
- Structure CLAUDE.md to maximize cache stability: put the most stable content (project description, architectural principles, team conventions) at the top, and the most variable content (current sprint notes, recent changes) at the bottom or in a separate injected file. Cache invalidation propagates from the point of change to the end of the document — a change near the top invalidates all caching below it.[^6]
- Monitor cache hit rates through the Anthropic API usage dashboard. A low cache hit rate on content that should be cached indicates a structural problem — frequently changing content near the top of the cached prefix, or session starts that do not consistently use the same context structure. Diagnosing and correcting a low cache hit rate is among the highest-ROI optimizations available.[^4]
- Do not cache session-specific context — task descriptions, temporary file contents, or conversation history from prior sessions. Caching dynamic content generates cache misses rather than hits, and the cache overhead (enabling caching on content that is not actually reused) adds marginal cost without benefit.

---

## Section 4: Estimating Session Costs Before Running Them

**Description:** Experienced teams develop pre-session cost intuition: before starting a Claude Code session, they can estimate whether it is a low-cost (sub-$1), medium-cost ($1–5), or high-cost (>$5) session based on the context size, model tier, expected turns, and output volume. This intuition does not require exact calculation — it requires a mental model calibrated to the team's actual session patterns. Sessions that exceed a mental cost threshold should be evaluated before running: is the expected output worth the expected spend?[^4]

The category of sessions with the highest cost risk are open-ended agentic sessions with broad scope, high model tier, and no defined termination criterion. These sessions are expensive not because any individual component is unusual, but because all cost drivers are simultaneously at their maximum: large context, many turns, long outputs, high model tier, and no forcing function to terminate before diminishing returns set in.

**Recommended Practice:**
- Before running any session that involves more than three files, agentic tool use, or Opus-tier model selection, do a quick cost estimate: approximate context size (CLAUDE.md + files + prior history) × expected turns × model input price, plus estimated output × model output price. This calculation takes 30 seconds and identifies sessions likely to generate $5+ spend before they run.[^4]
- Define a session termination criterion before starting the session: "This session is complete when [specific outcome] is achieved." Sessions without termination criteria tend to overrun because there is no forcing function to stop exploring. A defined criterion also helps evaluate whether the session delivered its expected value.
- For exploratory research sessions, timebox rather than scope: "Spend no more than 10 turns exploring this approach; if you have not reached a conclusion by turn 10, summarize what you have found and stop." Timebox constraints are a cost proxy in the absence of scope constraints.[^3]
- Track estimated vs. actual session cost for non-trivial sessions in the team's session log. Calibration improves with feedback; engineers who track their estimates become significantly better at cost estimation within a few months of practice.

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Token Mechanics | Train team on token mental model; prompt for code-first responses | Engineering team |
| Context History Costs | Establish `/compact` cadence at session phase transitions | Engineering team |
| Prompt Caching | Enable caching; restructure CLAUDE.md for cache stability | Architect |
| Pre-Session Estimation | Add cost estimate habit for multi-file and agentic sessions | Engineering team |

---

[^1]: Anthropic — "Models Overview," Anthropic API Documentation, 2026. https://docs.anthropic.com/en/docs/about-claude/models/overview
 Token pricing per model tier; input vs. output token cost ratios; tokenization rules and practical estimation heuristics for code and English text.

[^3]: Simon Willison — "How I Use LLMs: A Pragmatic 2026 Field Guide," simonwillison.net, February 2026. https://simonwillison.net/2026/Feb/how-i-use-llms/
 Output token cost implications of verbose prompting patterns; code-first vs. explanation-first prompt patterns; the cost of "think step by step" on low-complexity tasks.

[^4]: Nir Gazit (Traceloop) — "From Bills to Budgets: How to Track LLM Token Usage and Cost Per User," Traceloop Blog, 2025. https://www.traceloop.com/blog/from-bills-to-budgets-how-to-track-llm-token-usage-and-cost-per-user
 Input/output token ratio baselines for common session types; cache hit rate monitoring; pre-session cost estimation methodology; estimated vs. actual cost calibration tracking.

[^5]: Anthropic — "Context Window Management," Claude Code Documentation, 2026. https://code.claude.com/docs/en/context-window
 `/compact` command mechanics and per-turn cost reset; context window accumulation in long sessions; phase-boundary compaction as the recommended timing strategy.

[^6]: Anthropic — "Prompt Caching," Anthropic API Documentation, 2026. https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
 Prompt caching mechanics and economics; 90% cost reduction for cache hits; CLAUDE.md structure for cache stability; cache invalidation propagation rules.
