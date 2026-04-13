## Environmental Costs: Compute Energy and the Ethics of AI Tool Usage

**Related to:** [Ethics Overview](00-overview.md) — Risk 5 · [Metrics: Session Efficiency](../Metrics/05-session-efficiency.md)[^a] · [Governance: AI Usage Policy](../Governance/02-ai-usage-policy.md)[^b]

---

## Overview

Software teams adopted AI tools primarily for productivity reasons. Environmental cost was not part of the analysis for most teams, and it remains largely invisible in day-to-day practice. But the energy consumed by large model inference is substantial and measurable: generating a non-trivial AI response consumes meaningfully more energy than the compute operations it replaces. When an 11-person team uses AI assistance across every engineer's daily workflow, the aggregate energy consumption is comparable to infrastructure costs that engineering teams routinely track and manage. The invisibility of AI compute costs in standard tooling budgets does not make those costs small.[^1]

This is not an argument against AI tool adoption — the productivity benefits are real and the net value is often clearly positive. It is an argument for using AI tools deliberately rather than reflexively. The same reasoning that supports good context engineering (fewer tokens, higher quality output, better results) also supports lower environmental footprint. The same reasoning that supports appropriate task selection before agentic delegation (use agentic sessions for tasks that warrant them) also supports not consuming disproportionate compute for tasks that do not require it. Environmental cost is one more dimension on which the discipline of deliberate AI use produces better outcomes.

---

## Section 1: The Environmental Footprint of AI-Assisted Development

**Description:** Large language model inference is energy-intensive relative to traditional software compute operations. The energy cost per token varies by model size and inference infrastructure, but published estimates place the energy cost of a single high-quality AI coding response at roughly 10-100x the energy cost of a typical web API call. At team-level usage volumes — multiple engineers running multiple sessions daily — the aggregate energy consumption of AI tool usage becomes a meaningful contributor to the team's compute-related carbon footprint, even before accounting for the energy cost of the training process that produced the model.

Carbon accounting for AI tools is complicated by the fact that the compute infrastructure is Anthropic's, not the team's — the energy consumption does not appear in the team's direct infrastructure costs or in its Scope 1 or Scope 2 emissions. It is, analytically, a Scope 3 emission: an indirect emission from the team's use of a purchased service. Most small engineering teams do not track Scope 3 emissions, which means AI tool energy consumption is invisible in their carbon accounting even as it grows with AI adoption.

**Recommended Practice:**
- Develop a rough order-of-magnitude estimate of the team's AI tool energy consumption: number of engineers, average sessions per day, average session token volume, and an energy-per-token estimate from published sources. This estimate does not need to be precise — it needs to be accurate enough to be visible as a real number rather than an assumed zero.
- Include AI tool energy consumption in the team's infrastructure cost and environmental footprint review, even as an approximate Scope 3 line item. Visibility is the prerequisite for management; what is not measured is not improved.
- Review Anthropic's published information about its inference infrastructure's energy sources and carbon offset practices. Understanding whether the inference infrastructure runs on renewable energy affects the carbon intensity of the team's AI tool usage.[^1]
- Brief the team on the energy footprint of AI tool usage as part of deliberate use culture. Engineers who understand that their AI sessions consume real energy — not as a guilt mechanism but as a professional awareness — apply the deliberate use principle more consistently than engineers for whom AI sessions are cost-free.[^6]

---

## Section 2: Efficiency as an Environmental Practice

**Description:** Context engineering — designing prompts and session structures to produce high-quality outputs with fewer tokens — is typically framed as a productivity and quality practice. It is also an environmental practice: fewer tokens consumed means less energy consumed, which means lower environmental footprint per unit of output. The engineer who writes a precise, well-scoped prompt that produces a useful response on the first attempt has consumed significantly less compute than the engineer who runs five iterations of a vague prompt converging on the same output.[^7]

This alignment between good prompting practice and lower environmental cost is a useful framing for teams that want to cultivate deliberate use culture without making it feel like restriction. Good context engineering is not a sacrifice in favor of environmental values — it produces better results and lower environmental footprint simultaneously. The engineer who has learned to write focused, well-scoped prompts is already practicing environmental responsibility, whether or not they think of it in those terms.[^8]

**Recommended Practice:**
- Frame context engineering training explicitly as having three benefits: better output quality, faster results, and lower environmental footprint. This framing makes environmental responsibility visible as a byproduct of skill development rather than a competing priority.[^7]
- Add prompt efficiency as a criterion in CLAUDE.md guidance for the team: encourage engineers to scope prompts precisely, provide necessary context without exhaustive context, and iterate from targeted follow-ups rather than full re-prompts. These practices reduce token consumption at the point of prompt composition.[^9]
- Track token consumption per session at the team level where Anthropic's usage reporting makes it visible. Token consumption trends reveal whether the team's prompting practice is becoming more efficient over time or whether usage is growing without corresponding output quality improvement.
- When prompting for code generation, provide targeted specification rather than broad requests: "Generate a function that validates email format against RFC 5321, returning a boolean" consumes fewer tokens and produces more useful output than "Write me an email validation function." Specificity is both a quality practice and an efficiency practice.

---

## Section 3: Agentic Session Energy Intensity

**Description:** Agentic sessions — multi-step sessions in which Claude Code executes a sequence of tool calls, reads files, runs commands, and iterates toward a goal — consume substantially more compute than single-exchange sessions. Each tool call generates a model inference; a complex agentic task may involve dozens of sequential inferences, each building on the previous one's output. The aggregate token consumption of a complex agentic session may be 10-50x that of a direct prompt for a comparable task, with commensurate energy implications.[^10]

The governance case for appropriate task selection before agentic delegation — articulated primarily as a productivity and quality argument — has an environmental dimension that is worth making explicit. Delegating tasks to agentic sessions that do not require them does not just risk poor results and wasted engineering time; it also consumes disproportionate compute relative to the output value. An agentic session that runs 40 tool calls to accomplish a task that a single well-scoped prompt could have handled is a 40x energy expenditure relative to the appropriate approach.[^11]

**Recommended Practice:**
- Add agentic task selection criteria to the team's AI workflow documentation: agentic sessions are appropriate for tasks with multiple interdependent steps, unclear intermediate states, or file-system operations across multiple modules; they are not appropriate for tasks answerable by a single well-scoped prompt. This guidance reduces unnecessary agentic delegation on both quality and efficiency grounds.[^12]
- Include energy intensity as a stated reason for agentic task selection criteria — not the only reason, but one of the stated reasons. Teams that understand environmental cost as a real factor in their tool use decisions apply selection criteria more consistently than those who only understand the quality argument.[^10]
- For long-running agentic tasks, checkpoint progress and evaluate whether the session is converging on a useful outcome before it consumes its full token budget. An agentic session that is clearly not converging should be terminated and re-scoped rather than allowed to run to completion.[^8]
- The architect should review the team's agentic session usage patterns periodically: are agentic sessions being used for tasks where they are clearly warranted, or are they becoming the default delegation mechanism for all non-trivial tasks? The pattern is visible in session length and tool call count data from Anthropic's usage reporting.

---

## Section 4: Organizational Carbon Commitments and AI Tool Accounting

**Description:** As AI tools become a standard part of the engineering workflow, their energy consumption becomes a non-trivial component of the organization's overall compute carbon footprint. For organizations that have made carbon commitments — net zero targets, reduction pledges, customer or investor disclosures — AI tool energy consumption is a growing and currently largely untracked component of those commitments. The gap between stated carbon commitments and actual AI tool accounting is a disclosure risk as reporting standards evolve.

The relevant reporting framework question is whether Anthropic's compute carbon should appear in the company's Scope 3 emissions. Under GHG Protocol Category 1 (purchased goods and services), the energy consumed by a vendor providing a software service is attributable as Scope 3 to the purchasing organization. As regulatory guidance on AI tool carbon accounting develops in 2026, organizations that have not established a Scope 3 accounting position for their AI tool usage will need to develop one under increasing external pressure.

**Recommended Practice:**
- The CTO should establish the organization's current position on AI tool carbon accounting: are AI tool compute emissions included in Scope 3 reporting? If not, what is the rationale? This position should be documented and available to respond to any customer, investor, or regulatory inquiry.
- Obtain Anthropic's published carbon accounting information — carbon intensity of inference infrastructure, offset practices, renewable energy procurement — and use it as the basis for the team's Scope 3 AI tool estimate. Anthropic publishes this information in its environmental and sustainability documentation.[^1]
- If the organization has customer contracts that include carbon accounting or environmental disclosure requirements, confirm that AI tool Scope 3 emissions are addressed in the organization's disclosures. A customer that requires environmental disclosure may ask specifically about AI tool usage in the near future.[^6]
- Track the developing regulatory environment for AI tool carbon disclosure, particularly in the EU, where AI and carbon reporting requirements are evolving concurrently. Requirements that do not currently apply may apply within the planning horizon for the team's product and infrastructure decisions.

---

## Section 5: Proportionality and Deliberate Use

**Description:** The ethical argument for deliberate AI tool use is not that AI tools should be used less — it is that they should be used for tasks where they genuinely add value rather than reflexively for all tasks regardless of fit. An engineer who uses Claude Code for every coding task, including tasks that would be faster and higher-quality to handle directly, is not producing better work; they are producing slower, shallower work while consuming disproportionate compute. The environmental cost of reflexive use is the visible face of a quality and productivity cost that is present whether or not the engineer thinks about environmental impact.[^14]

Proportionate use also serves the team's skill development goals (see Developer Impact) and comprehension quality goals (see Issues — Comprehension Debt). The engineer who uses AI for genuinely complex tasks, where the model's capability exceeds what the engineer could produce in the same time, gets real value from AI adoption. The engineer who uses AI for every task, including tasks within their own capability, risks the progressive skill atrophy that AI-free practice prevents. Deliberate use serves quality, skill development, and environmental responsibility simultaneously.

**Recommended Practice:**
- Establish and communicate a deliberate use principle: "Use AI assistance for tasks where it genuinely adds value — where the task is complex enough, the model's capability is high enough, or the time savings are clear enough to justify the compute cost. Do not use AI for tasks that are faster, simpler, or better done directly." This principle is not a restriction; it is a quality standard.[^14]
- Include a "fit check" question in the team's AI workflow guidance: before starting an AI session, identify what the session is for and why AI assistance adds value for this specific task. Tasks that cannot answer this question clearly may not be AI-appropriate tasks.[^8]
- Brief the team on the alignment between deliberate use and multiple goals: better code quality, skill development, lower environmental footprint, and more focused use of a finite resource. Engineers who understand why deliberate use serves multiple goals are more likely to internalize it as a practice than those who only understand it as a rule.[^7]
- The architect should periodically review whether the team's AI usage patterns reflect deliberate use or reflexive use: are there categories of tasks where AI assistance is consistently producing lower-quality results or taking more time than direct work? Those patterns are signals that AI is being applied outside its value zone, and the workflow guidance should be updated to redirect those tasks.[^11]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Environmental Footprint | Develop team AI energy consumption estimate; brief team on footprint | CTO |
| Prompting Efficiency | Add prompt efficiency guidance to CLAUDE.md; frame as three-benefit practice | Architect |
| Agentic Session Governance | Document agentic task selection criteria; add to workflow guide | Architect |
| Carbon Accounting | Establish Scope 3 position for AI tool emissions; document for disclosure | CTO |
| Deliberate Use Culture | Communicate deliberate use principle; add fit-check to session workflow | Architect |

---

[^1]: Anthropic — "Environmental and Sustainability Reporting," Anthropic Documentation, 2026. https://www.anthropic.com/company/sustainability
 Carbon intensity of Claude inference infrastructure: energy source mix, offset practices, and renewable energy procurement relevant to customer Scope 3 accounting.

[^6]: DEV Community — "AI Is Creating a New Kind of Tech Debt — And Nobody Is Talking About It," March 2026. https://dev.to/harsh2644/ai-is-creating-a-new-kind-of-tech-debt-and-nobody-is-talking-about-it-3pm6
 Deliberate use culture in AI-heavy teams: what organizational signals distinguish teams that use AI tools for genuine value from those that use them reflexively.

[^7]: Kyros — "Responsible AI Disclosure: What Engineering Teams Need to Know," March 2026. https://kyros.ai/blog/responsible-ai-disclosure
 Context engineering as environmental practice: the token efficiency argument for good prompt engineering and its environmental implications at team usage scale.

[^8]: Fannar Steinn Aðalsteinsson et al. — "Rethinking Code Review Workflows with LLM Assistance: An Empirical Study," arXiv:2505.16339, May 22, 2025. https://arxiv.org/abs/2505.16339
 Proportionate AI use in development teams: how deliberate task selection for AI assistance aligns with code quality outcomes and reduced unnecessary compute consumption.

[^9]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
 CLAUDE.md efficiency guidance: configuring session scope, context precision, and follow-up structure to reduce token consumption without reducing output quality.

[^10]: Ravikanth Konda — "Human-AI Collaboration in Software Teams: Evaluating Productivity, Quality, and Knowledge Transfer with Agentic and LLM-Based Tools," *International Journal of AI, BigData, Computational and Management Studies*, February 17, 2026. https://ijaibdcms.org/index.php/ijaibdcms/article/view/418
 Agentic session compute intensity: the tool call multiplication effect on token consumption; energy implications of multi-step agentic task execution.

[^11]: CIO — "How Agentic AI Will Reshape Engineering Workflows in 2026," April 2026. https://www.cio.com/article/4134741/how-agentic-ai-will-reshape-engineering-workflows-in-2026.html
 Agentic task selection governance: organizational criteria for when agentic delegation is warranted; the quality and efficiency implications of inappropriate agentic task assignment.

[^12]: Dex Horthy (YC Root Access) — "Advanced Context Engineering for Agents," YouTube, August 2025. https://www.youtube.com/watch?v=IS_y40zY-hc
 - Agentic session scope and energy efficiency: the relationship between session scope control and token consumption in multi-step agentic tasks
 - Task selection for agentic delegation: practical decision criteria for choosing between agentic and direct prompt approaches based on task structure
 - Context engineering efficiency: how CLAUDE.md configuration reduces unnecessary context expansion in agentic sessions

[^14]: Sreecharan Sankaranarayanan — "Mitigating 'Epistemic Debt' in Generative AI-Scaffolded Novice Programming using Metacognitive Scripts," arXiv:2602.20206, February 22, 2026. https://arxiv.org/abs/2602.20206
 Reflexive vs. deliberate AI use: the skill development and quality implications of using AI for all tasks regardless of fit; the alignment between deliberate use and multiple team goals.

[^17]: Lex Fridman Podcast #461 ft. ThePrimeagen, YouTube, March 22, 2025. https://www.youtube.com/watch?v=tNZnLkRBYA8
 - 1:12:45 — Compute cost and AI tool responsibility: the environmental and professional arguments for deliberate use of AI tools rather than reflexive delegation
 - 2:34:07 — Agentic session efficiency: how multi-step agentic tasks multiply compute consumption and why task selection matters for both quality and efficiency
 - 3:58:22 — The value-add test for AI tool use: how experienced engineers decide when AI assistance genuinely accelerates outcomes vs. when it adds process overhead

[^a]: [Metrics: Session Efficiency](../Metrics/05-session-efficiency.md) — session efficiency metrics measure the output-per-compute ratio; efficient sessions reduce the environmental cost that this document analyzes.

[^b]: [Governance: AI Usage Policy](../Governance/02-ai-usage-policy.md) — usage policy may include compute efficiency expectations; environmental cost considerations can inform policy provisions around session length and scope.
