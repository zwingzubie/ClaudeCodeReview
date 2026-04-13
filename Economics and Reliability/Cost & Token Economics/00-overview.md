## Overview

For a team of 11 using Claude Code at scale, the cost of AI-assisted development is not a rounding error on the infrastructure budget — it is a variable expense that scales directly with the team's most productive sessions. A week of heavy agentic usage, a few poorly scoped sessions that re-send the entire codebase as context, or a consistent failure to use the right model for the right task can generate API spend that materially exceeds expectations. Teams that treat cost management as a finance problem rather than an engineering discipline tend to discover their spend profile only in retrospect — after an unexpected invoice, an engineering manager's quarterly review, or a rate-limit interruption during a critical sprint.[^1]

The economics of AI coding tools in 2026 are structured around three levers: model selection, context size, and prompt caching. These levers interact. A team that consistently chooses Opus for tasks where Haiku would suffice is paying a 20× per-token premium. A team that consistently passes 100,000 tokens of context when 15,000 would answer the question is spending six times as much per session. A team that has not enabled prompt caching on repeated CLAUDE.md and codebase reads is paying full price for content it has already sent many times. Across a team of 11, the compounding effect of unmanaged defaults in all three areas can produce monthly spend two to three times higher than an equivalent team that has calibrated its tooling.[^2]

The five areas documented below are the structural response to that compounding. They address the mechanics of token pricing, the criteria for model selection, the techniques for context optimization, the governance framework for team budgeting, and the tooling for monitoring and alerting on spend. Each area is individually actionable; together they provide the cost discipline that makes AI-assisted development economically sustainable at team scale.

---

## Cost Area 1: Understanding Token Costs

**Description:** Claude Code's cost is denominated in tokens — the units into which the API divides all text, code, and context before processing. Input tokens (everything sent to the model: the prompt, CLAUDE.md, injected files, conversation history) and output tokens (everything the model generates: code, explanations, plans) are priced separately, with output tokens typically priced two to five times higher than input tokens depending on the model. Engineers who do not understand the token model have no basis for intuition about which sessions are expensive and which are not.[^3]

The cost structure of a Claude Code session is not primarily determined by the engineer's typed prompt — it is determined by context. A CLAUDE.md file, injected spec files, conversation history, and automatically included codebase content collectively constitute the input context for every session turn. In a long session with a 10,000-token CLAUDE.md, three injected files totaling 8,000 tokens, and 15,000 tokens of conversation history, the engineer's 200-token prompt is less than 1% of the input cost per turn. That proportion inverts the intuition that "typing less saves money" and explains why context management — not prompt brevity — is the primary cost driver for teams using Claude Code seriously.

**Proposed Solution:**
- Train the engineering team on a practical token mental model: one page of English text is approximately 750 tokens; one page of code is approximately 500–700 tokens. This gives engineers a working intuition for context size without requiring them to count tokens explicitly.[^3]
- Enable prompt caching for all content that is sent repeatedly across sessions: CLAUDE.md, project specification files, architecture documents, and boilerplate context. Prompt caching reduces the per-token cost of cached content by approximately 90% for cache hits, and is the highest-leverage single optimization available for teams with consistent session structures.[^2]
- Review the token economics of the team's most common session types: estimate the typical input context, number of turns, and output volume for each session pattern. This exercise identifies which session patterns are disproportionately expensive relative to their output value.
- Include token usage in the team's session log (see Metric 4: Session and Context Efficiency). Usage data accumulated over a quarter reveals which task types, which engineers, and which codebase areas generate the most cost — providing the denominator for cost-per-value analysis.

---

## Cost Area 2: Model Selection Strategy

**Description:** In 2026, the Claude model family spans a significant capability and cost range. Haiku is designed for fast, cost-efficient tasks; Sonnet balances capability and cost for the majority of engineering work; Opus is the highest-capability model, appropriate for the most complex reasoning tasks. The per-token price difference between Haiku and Opus on the input side is approximately 20×; on the output side it is similar. A team that defaults to Opus for every session regardless of task complexity is not getting better results on simple tasks — it is paying a 20× premium for capability it does not need.[^5]

The routing problem — assigning the right model to the right task — is not merely a cost problem. It is also a quality problem in the opposite direction: using Haiku for complex architectural reasoning or for tasks requiring subtle judgment produces worse output than Sonnet or Opus, and the cost savings are negated by additional correction cycles. The team's model selection policy should be calibrated to match model capability to task complexity across the full range of tasks the team actually runs, not default to either extreme.[^6]

**Proposed Solution:**
- Define a three-tier task classification for the team: Tier 1 (Haiku) — boilerplate generation, test scaffolding, documentation formatting, simple code edits with clear specifications; Tier 2 (Sonnet) — feature implementation, bug diagnosis, code review, moderate refactoring; Tier 3 (Opus) — architectural planning, complex debugging across multiple systems, security analysis, tasks requiring extended reasoning over large codebases.[^5]
- Set Sonnet as the team's default model in Claude Code configuration. Defaulting to Sonnet rather than Opus captures the majority of tasks at a 3–5× cost reduction while retaining full capability for the most common engineering work. Engineers escalate to Opus explicitly when they have a specific reason.[^6]
- Document model escalation criteria in CLAUDE.md so that the rationale is available to engineers without requiring consultation: "Use Opus when the task involves architectural decisions affecting multiple systems, when debugging requires reasoning across more than three interacting components, or when the task has failed two or more correction cycles with Sonnet."[^6]
- Review model selection choices quarterly as part of the engineering health review. If the team's Opus usage is concentrated in a small set of recurring task types, those types are candidates for better CLAUDE.md context that reduces their complexity and allows them to be handled at Sonnet tier.

---

## Cost Area 3: Context Optimization

**Description:** Context optimization is the practice of sending the minimum context required for a high-quality response, structured to maximize prompt cache hit rates, and pruned aggressively as sessions progress. It is the primary engineering discipline for controlling per-session cost because it directly reduces the token count of every session turn — not just the final output, but every intermediate message in the conversation.

The failure mode context optimization addresses is context accumulation: the gradual inflation of session context as conversation history grows, as engineers inject additional files to resolve errors, and as the model's responses become longer in response to perceived ambiguity. An unmanaged session that starts with 20,000 tokens of context may reach 80,000 tokens of context by turn 15 — four times the per-turn cost, with diminishing returns on quality as irrelevant context dilutes the model's attention. The `/compact` command exists specifically for this failure mode, but engineers who do not understand the cost model have no reason to use it proactively.[^7]

**Proposed Solution:**
- Establish a team norm for proactive `/compact` usage: use `/compact` when a session has completed a coherent phase of work and is about to begin a different sub-task, or when the conversation history has grown to the point where early context is no longer relevant to the current task. Do not wait until quality degrades — compact before quality degrades.[^7]
- Structure CLAUDE.md for maximum cache efficiency: put the most stable, general content at the top (project description, team conventions, architectural principles) and the most specific, task-variable content at the bottom. Prompt caching rewards consistency in the prefix of the context; changing content near the top of CLAUDE.md invalidates the cache for all content below it.[^2]
- Use targeted file injection rather than whole-directory injection. When Claude Code needs to understand a module, inject the specific files relevant to the task rather than the entire module directory. The quality difference on a focused task is small; the token difference can be 5–10× for a large module.
- Before starting a session, define its scope explicitly: what files are in scope, what questions need to be answered, what the session will not address. A scoped session with a clear termination criterion accumulates less irrelevant context than an open-ended exploratory session and produces more predictable cost.

---

## Cost Area 4: Budget Governance

**Description:** Without explicit budget governance, AI development spend on a team of 11 is determined by individual engineer behavior, session defaults, and the volume of work in a given sprint — none of which are directly controlled or even visible to the people responsible for the team's operating budget. The structural solution is to treat Claude Code API spend as a managed line item: budgeted by sprint, tracked by engineer, reviewed by the architect, and subject to the same approval workflow as other infrastructure spend above a threshold.[^1]

Budget governance is not about restricting engineer access to AI tools — it is about creating the visibility that allows the team to make deliberate decisions about where AI spend is generating value. A sprint where $800 of Claude Code spend produced a two-week feature ahead of schedule is a different outcome than a sprint where $800 of spend produced three rounds of rework and a late delivery. The difference is not visible without both spend data and output quality data; budget governance creates the spend side of that comparison.[^8]

**Proposed Solution:**
- Set a monthly API spend budget at the team level, allocated by sprint. Establish three budget tiers: normal sprint (baseline allocation), high-AI sprint (1.5× baseline, requires architect approval before the sprint begins), and incident response (uncapped for time-critical production issues, reviewed post-incident).[^8]
- Configure API key usage limits at the Anthropic console level for each engineer's key where possible, with a soft alert threshold at 70% of monthly allocation and a hard limit at 110% (allowing some overrun with notification). Keys that exceed the soft threshold trigger a brief review of session logs to identify whether the spend is generating proportionate value.[^1]
- Assign cost stewardship to the architect: a monthly 15-minute cost review covering total spend, per-engineer breakdown, model distribution, and trend vs. prior month. The steward is not a cost police function — it is an early-warning function that identifies when usage patterns have drifted from the team's calibrated defaults.[^8]
- Include AI API spend in the quarterly engineering health review alongside other infrastructure costs. Present cost-per-feature-delivered as the outcome metric rather than raw spend: the question is not whether $X was spent, but whether $X generated proportionate engineering output.

---

## Cost Area 5: Cost Monitoring and Alerting

**Description:** Manual budget governance is necessary but insufficient at the frequency with which individual sessions can accumulate spend. A single agentic session with broad scope, many turns, and Opus-tier model selection can generate more spend in one hour than the team's average daily budget. Without near-real-time monitoring and automated alerting, the first indication of an anomalous session is the end-of-month invoice — well after any corrective action is possible.[^9]

The monitoring infrastructure for Claude Code spend is available through the Anthropic API usage dashboard and through third-party tools that aggregate API cost data alongside other cloud infrastructure costs. The alerting layer — thresholds that notify the architect or team lead when usage is trending above plan — requires configuration, but is straightforward to implement once the budget baselines are established.[^1]

**Proposed Solution:**
- Configure usage export from the Anthropic console to a shared team dashboard (cost management tools like Metronome, or a simple spreadsheet updated daily from the Anthropic usage API). The dashboard should show daily spend, cumulative monthly spend vs. budget, model distribution, and top sessions by cost.[^9]
- Set automated alerts at the Anthropic console level: alert at 70% of monthly budget (planning trigger) and 90% of monthly budget (restrict trigger). The planning alert initiates a mid-month review; the restrict alert triggers a brief hold on non-critical agentic sessions until the architect reviews current usage.[^1]
- Instrument the team's Claude Code hooks to log session token counts and model selections to a shared log file. This gives the architect cost-attribution data at the session level — not just total API spend — and identifies which session types or engineers are driving anomalous spend.[^9]
- Review the monitoring and alerting configuration quarterly: budget baselines change as the team's AI adoption matures, and alert thresholds calibrated for month one may be either too tight or too loose by month six. Quarterly recalibration keeps the alerting layer useful rather than habituated.

---

## Summary of Recommended Actions

| Cost Area | Immediate Action | Owner |
|---|---|---|
| Understanding Token Costs | Enable prompt caching; train team on token mental model | Architect |
| Model Selection Strategy | Set Sonnet as default; document Opus escalation criteria in CLAUDE.md | Architect |
| Context Optimization | Establish `/compact` norms; audit CLAUDE.md for cache-unfriendly structure | Backend lead |
| Budget Governance | Set monthly budget by sprint tier; configure per-engineer soft limits | Architect + CTO |
| Cost Monitoring and Alerting | Configure usage dashboard; set 70%/90% automated alerts | Architect |

---

[^1]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
 API key management and spend visibility; budget governance as a prerequisite for sustainable AI adoption at team scale; session configuration defaults and their cost implications.

[^2]: Anthropic — "Prompt Caching," Anthropic API Documentation, 2026. https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
 Prompt caching mechanics and hit rate optimization; CLAUDE.md structure for maximum cache efficiency; the 90% cost reduction for cache hits and its implications for teams with consistent session structures.

[^3]: Anthropic — "Models Overview," Anthropic API Documentation, 2026. https://docs.anthropic.com/en/docs/about-claude/models/overview
 Token pricing by model tier; input vs. output token cost ratios; the practical token mental model for engineering teams; model capability and cost comparison across the Claude family.

[^5]: Simon Willison — "How I Use LLMs: A Pragmatic 2026 Field Guide," simonwillison.net, February 2026. https://simonwillison.net/2026/Feb/how-i-use-llms/
 Three-tier task classification for model selection; the 20× cost differential between Haiku and Opus; calibrating model selection to task complexity rather than defaulting to maximum capability.

[^6]: Anthropic — "2026 Agentic Coding Trends Report," Anthropic, 2026. https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf
 Model selection as both a cost and quality decision; escalation criteria for Opus-tier tasks; team-level model policy as a governance artifact rather than individual engineer preference.

[^7]: Anthropic — "Managing Long Sessions," Claude Code Documentation, 2026. https://code.claude.com/docs/en/managing-long-sessions
 `/compact` command mechanics and timing; context accumulation as a cost driver in long sessions; the quality-cost tradeoff of unmanaged context growth; session scope definition as a cost control.

[^8]: WorkOS — "AI Development Costs in 2026: What Engineering Teams Are Actually Spending," WorkOS Blog, January 2026. https://workos.com/blog/ai-development-costs-2026
 Sprint-level budget allocation patterns; cost-per-feature-delivered as the governance metric; three-tier budget structure for normal, high-AI, and incident-response sprints; the spend-visibility gap in teams without budget governance.

[^9]: Stephanie Hurlburt — "Tracking LLM Costs Across a Small Engineering Team," Stephanie Hurlburt's Blog, March 2026. https://stephanehurlburt.com/blog/tracking-llm-costs
 Per-session cost attribution via hook instrumentation; dashboard configuration for daily spend tracking; anomaly detection patterns for agentic session cost spikes; quarterly recalibration of alert thresholds.

[^10]: Sabrina Ramonov — "CLAUDE CODE FULL COURSE," YouTube, February 17, 2025. https://www.youtube.com/watch?v=fYX6hHC9FhQ
 - Budget governance workflow: how to configure the Anthropic console for per-engineer spend tracking
 - Session cost estimation: pre-session context sizing before running expensive agentic workflows
 - `/compact` timing: when to compact within a session to avoid the context inflation cost curve
