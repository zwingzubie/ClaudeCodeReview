## Quarterly Engineering Health Review: Strategic Governance and CTO Decision Authorities

**Related to:** [Governance Overview](00-overview.md) — Policy 5: Quarterly Engineering Health Review · [Metrics: Team Health Dashboard](../Metrics/02-health-dashboard.md)[^a] · [Documentation: Architecture Decision Records](../Documentation/01-architecture-decision-records.md)[^b] · [Governance: Compliance and Audit](06-compliance-and-audit.md)[^c] · [Issues: Velocity Governance](../Issues/08-velocity-governance.md)[^d]

---

## Overview

Monthly practice reviews are operational — they track whether the team is following the practices it has committed to following. They catch drift, surface individual incidents, and produce adjustments to process. What monthly reviews cannot produce is strategic visibility: whether the direction of drift over the past six months is toward better or worse outcomes, whether the team's AI adoption is producing sustainable velocity improvement or accumulating a rework liability that will surface as delivery failures in a future quarter, or whether the governance decisions made in Q1 actually produced the improvements they were designed to produce. That visibility requires a quarterly cadence with a different analytical frame.

The quarterly engineering health review is the CTO's primary mechanism for exercising strategic oversight of AI-assisted engineering. It is not a longer version of the monthly practice review — it is a different artifact, operating at a different time horizon, with different decision authorities attached to it. The monthly review asks: are we following our practices? The quarterly review asks: are our practices producing good engineering outcomes, and are our governance decisions calibrated correctly to support them? Only the quarterly review can surface the kind of trend signals that DORA's research has captured: teams managing AI governance actively saw 7.2% delivery stability improvement, while teams without active governance saw equivalent degradation over the same period.[^2]

---

## Section 1: The Strategic vs. Operational Distinction

**Description:** The strategic vs. operational distinction in governance review is not just about cadence — it is about the analytical unit. Operational reviews examine individual practices and individual incidents: did this PR meet the review standard? was this sprint's AI-primary percentage within bounds? did this violation get handled correctly? Strategic reviews examine aggregate patterns and trend directions: is the team's AI-generated code quality improving or degrading over time? is technical debt accumulation accelerating? are delivery outcomes improving as AI adoption matures, or are they plateauing in a way that suggests the current governance model has reached its effectiveness ceiling?

Monthly data is too noisy for strategic signal extraction. A single sprint with unusual characteristics — a team member out sick, an unusually complex story, a production incident that consumed review capacity — can make a month look anomalous without reflecting any underlying trend. Six-month trend data smooths this noise and reveals the direction the team is actually moving. DORA's 2025 research found that delivery stability differences between teams with active AI governance and those without were not visible in any single month's data, but were clearly visible in six-month rolling averages — which is precisely the analytical unit the quarterly review is designed to produce.[^2]

**Recommended Practice:**
- The quarterly health review requires six-month chart data for all tracked metrics, not just the most recent quarter. A six-month chart shows trend direction, rate of change, and whether interventions from prior quarters produced the intended effects. Quarterly point-in-time data without trend context produces decisions that respond to noise rather than signal.
- Distinguish the strategic review clearly from the monthly practice review in the team calendar and in the review documents: different format, different agenda, different decision authorities. Teams that conflate the two produce reviews that are neither operationally useful nor strategically informative.
- The CTO is the primary decision-maker at the quarterly review, not an attendee. The architect presents the data and analysis; the CTO interprets strategic signals and makes the governance decisions that require CTO authority (Section 4). Reviews where the CTO attends but does not exercise decision authority are not functioning as strategic governance.[^4]
- Prepare the quarterly review data package at least three days before the review meeting: six-month metric charts, governance effectiveness evaluation (Section 3), and decision items with pre-prepared cost-benefit analysis. The review meeting is for interpretation and decision, not for data collection. Same-day data preparation produces poor strategic analysis.[^5]

---

## Section 2: The Quarterly Review Presentation Structure

**Description:** The quarterly review presentation is a four-section document that moves from summary to data to interpretation to recommendation. The executive summary gives the CTO a two-paragraph assessment of engineering health status and the most important decisions required. The metric trends section presents six-month charts for each tracked metric with brief annotations at inflection points. The interpretation and risk assessment section translates metric findings into delivery risk language — not "cyclomatic complexity increased" but "complexity growth in the payment module creates a 40% higher estimated defect density for the next major change." The governance recommendations section proposes specific changes with cost-benefit analysis attached.[^6]

Framing metric findings in terms of delivery risk rather than technical quality is not spin — it is the appropriate translation for a decision-making audience. The CTO's governance decisions at the quarterly review are fundamentally risk management decisions: is the current trajectory acceptable, or does it require intervention that consumes velocity in the short term to prevent larger problems in a later quarter? Technical quality metrics are inputs to that risk assessment, not the output. A presentation that reports only technical metrics without translating them into delivery risk outcomes leaves the CTO to perform that translation without the context the architect has — and produces worse decisions.[^7]

**Recommended Practice:**
- Use the standard four-section structure for every quarterly review: executive summary (two paragraphs), metric trends (six-month charts for each metric), interpretation and risk assessment (delivery risk framing), governance recommendations (specific proposals with cost-benefit analysis). Structural consistency across quarters makes trend comparison easier and prevents each review from requiring a fresh orientation.[^6]
- The executive summary must include a one-sentence engineering health status (Green / Yellow / Red with a brief rationale) and the top decision item requiring CTO attention. The CTO should be able to read the executive summary in two minutes and understand what the review requires of them before opening the detailed sections.[^7]
- Metric annotations are required at inflection points in the six-month charts: what happened in the sprint or month where a metric changed direction? Was it a policy change, a staffing change, a major delivery event, or unexplained? Inflection annotations prevent the CTO from misattributing metric movements to the wrong cause.
- The governance recommendations section must propose specific, actionable changes — not "consider improving review quality" but "raise the AI-primary code secondary review requirement from security-critical modules to include all modules over 200 lines, estimated cost: 0.5 hours per sprint, estimated risk reduction: 25% in review-escaped defect rate based on Q1/Q2 data." Vague recommendations produce no decisions; specific recommendations with cost-benefit analysis produce approvals or denials with reasoning.[^5]

---

## Section 3: Governance Effectiveness Evaluation

**Description:** The governance effectiveness evaluation is the accountability loop at the heart of the quarterly review: did last quarter's governance decisions produce the intended improvements? This is the question that distinguishes active governance from policy theater. A team that makes governance decisions without ever evaluating whether they worked is not governing — it is administering rules. Active governance requires closing the feedback loop: we made a specific decision, we expected a specific outcome, here is what actually happened, and here is what that tells us about whether the decision was correct and whether the governance model is working.

Governance ROI is a legitimate concept. Policy enforcement, sprint planning gates, review requirements, and escalation procedures all consume engineering time. The question is whether the time investment produces better outcomes than the counterfactual. The governance effectiveness evaluation quantifies this — imperfectly, but usefully. If the Q2 decision to require spec.md for all AI-primary tasks consumed an estimated 0.5 story points per AI-primary story in specification effort, but produced a 40% reduction in AI-primary review round counts (each round at approximately 1 story point of combined engineer time), the ROI is strongly positive. If spec.md effort increased without review round reduction, the policy needs recalibration.

**Recommended Practice:**
- For each significant governance decision made in the prior quarter, prepare a one-page effectiveness evaluation: what was the decision, what outcome was expected, what metrics tracked the intended improvement, what did those metrics show, and what is the recommended adjustment (keep, modify, or retire the policy change).
- When a governance intervention produces no measurable improvement after one quarter, the response is not automatic: first determine whether the metric being tracked is actually the right leading indicator for the intended outcome. If the metric is appropriate and shows no improvement, escalate to the CTO at the quarterly review for a decision about whether to continue, modify, or retire the intervention.
- Measure governance ROI by comparing estimated policy compliance cost (engineer time per sprint) against estimated benefit (rework reduction, defect escape reduction, review cycle time reduction). These estimates will be approximate, but the order of magnitude is usually clear enough to support decisions. A governance practice that costs four hours per sprint and produces no measurable benefit is a candidate for simplification.[^2]
- Document governance effectiveness evaluations in the governance changelog. Over multiple quarters, the evaluation record reveals which governance practices have demonstrated sustained value and which have produced diminishing returns — which is the data needed for the annual governance model review that connects to annual planning (Section 5).[^4]

---

## Section 4: CTO Decision Authorities at the Quarterly Review

**Description:** The quarterly review is the primary venue for decisions that require CTO authority. Distinguishing CTO-authority decisions from architect-authority decisions matters because conflating them either overloads the CTO with operational decisions that do not require their involvement, or leaves strategic decisions without the authority level they require. The quarterly review is explicitly designed to surface the strategic decisions and present them with the analysis needed for a well-informed CTO decision, rather than asking the CTO to evaluate proposals without adequate context.[^10]

CTO-authority decisions at the quarterly review include: changes to sprint AI-primary percentage caps (which affect team-wide delivery commitments and stakeholder expectations), new tooling acquisition above a defined cost threshold (which affects vendor relationships and compliance obligations), policy category changes (which affect the governance model's scope and coverage), and resource allocation for engineering health investments (which requires authority over headcount and budget). Each of these decisions has consequences beyond the engineering team that make CTO ownership appropriate rather than architect delegation.[^11]

**Recommended Practice:**
- Prepare each CTO-authority decision item as a structured proposal: the decision required, the options considered, the cost-benefit analysis for each option, the architect's recommendation, and the data from the quarterly review that informs the recommendation. The CTO decides; the architect analyzes. Decision proposals that arrive without pre-analysis require the CTO to perform the analysis at the meeting, which produces worse decisions and longer meetings.[^10]
- For sprint AI-primary cap changes, present the full six-month data: current cap utilization trend, mid-sprint triage frequency, review cycle time correlation with AI-primary percentage, and the proposed new cap with the rationale. A cap increase that is not supported by improving review quality metrics is not ready for CTO approval regardless of velocity pressure.[^11]
- For new tooling acquisition decisions, include vendor security documentation review status (see Governance/06), estimated compliance implications, and the new capability authorization status (see Governance/04). A tooling acquisition decision at the quarterly review should arrive with pre-work completed, not as a discovery that additional analysis is required.[^5]
- Document every CTO decision made at the quarterly review — approval, denial, or deferral — with the reasoning. Documented CTO decisions create accountability for the governance direction and provide the context needed to evaluate governance effectiveness in subsequent quarterly reviews.[^4]

---

## Section 5: Connecting Quarterly Findings to Annual Planning

**Description:** The quarterly review is not an isolated event — it is a data collection point in a continuous governance cycle that has an annual cadence at its outer boundary. Annual planning decisions about engineering capacity, technical debt investment, tooling budget, and team structure should be informed by what the quarterly reviews revealed over the prior year. A team that produces four quarterly reviews per year and then ignores them in annual planning is extracting less than half the governance value from the quarterly review process.[^12]

The specific governance findings that translate to annual planning inputs are: persistent technical debt patterns that require roadmap investment (rather than sprint-level remediation), tooling gaps that require budget allocation, team capability gaps that require hiring or training investment, and governance model changes that require policy infrastructure investment. Each of these surfaces in quarterly review data before it becomes a planning emergency — the quarterly review is the early warning system that gives annual planning the time and information needed to address issues deliberately rather than reactively.[^13]

**Recommended Practice:**
- Produce an annual governance summary at year-end that aggregates the four quarterly reviews: what were the persistent trends, what governance decisions were made, what was their effectiveness, and what patterns appeared in the escalation and override logs that suggest structural issues? This summary is the primary engineering input to the annual planning cycle.[^12]
- Technical debt items discovered or quantified in quarterly reviews that exceed a defined complexity or risk threshold become roadmap candidates with a standard template: what the debt is, what the quarterly data shows about its growth trajectory, what the estimated remediation cost is, and what the estimated cost of inaction is over the next 12 months.[^13]
- The CTO uses quarterly governance findings in capacity planning decisions: a team that is consistently above the AI-primary review capacity threshold may need a reviewer headcount increase; a team with persistent CLAUDE.md compliance gaps may need a dedicated governance engineering investment; a team with expanding agentic use may need architect time allocation for agentic session oversight.[^11]
- Schedule the annual governance model review as a separate event from Q4 quarterly review, with the four quarterly reports as inputs. The annual review asks the strategic meta-question: is the governance model itself — its structure, its overhead, its coverage — still appropriate for the team's current AI maturity and work patterns? Teams that started with conservative governance models should assess whether their maturity warrants simplification; teams that have experienced governance failures should assess whether more structure is required.[^2]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Six-Month Chart Data | Configure metric tracking to produce six-month rolling charts for all KPIs | Architect |
| Four-Section Presentation Structure | Create quarterly review presentation template with required sections | Architect |
| Delivery Risk Framing | Translate each metric trend into a delivery risk statement before the review meeting | Architect |
| Three-Day Data Preparation Lead Time | Block three days before each quarterly review for data package preparation | Architect |
| Governance Effectiveness Evaluations | Prepare one-page effectiveness evaluation for each prior-quarter governance decision | Architect |
| CTO Decision Proposals | Prepare structured proposal for each CTO-authority decision with options and analysis | Architect |
| Annual Governance Summary | Produce year-end aggregate of four quarterly reviews as annual planning input | Architect |
| Strategic Review Attendance | Attend quarterly review as primary decision-maker, not observer | CTO |
| Sprint AI-Primary Cap Change Decisions | Approve or deny cap changes based on six-month review quality correlation data | CTO |
| New Tooling Acquisition Decisions | Make tooling acquisition decisions at quarterly review with pre-completed compliance analysis | CTO |
| Technical Debt Roadmap | Convert qualifying quarterly review debt findings to annual roadmap candidates | CTO |
| Annual Governance Model Review | Facilitate annual meta-review of governance model structure and coverage | CTO |

---

[^2]: DORA / Google Cloud — "DORA 2025 Accelerate State of DevOps Report," 2025. https://dora.dev/research/2025/
 7.2% delivery stability improvement for teams with active AI governance vs. equivalent degradation without; six-month rolling average as the analytical unit where governance differences become visible.

[^4]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
 CTO as primary decision-maker at quarterly reviews; governance effectiveness documentation in the changelog; annual governance model review timing.

[^5]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
 Quarterly review preparation timeline; tooling acquisition pre-work requirements; decision proposal completeness as a prerequisite for CTO decision quality.

[^6]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
 Four-section presentation structure: executive summary through governance recommendations; structural consistency across quarters for trend comparison.

[^7]: CIO — "How Agentic AI Will Reshape Engineering Workflows in 2026," April 2026. https://www.cio.com/article/4134741/how-agentic-ai-will-reshape-engineering-workflows-in-2026.html
 Delivery risk framing for technical metric findings; the translation from technical quality metrics to delivery outcome language appropriate for CTO decision-making.

[^10]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
 CTO-authority decision categories; structured decision proposals as a prerequisite for high-quality quarterly review decisions; the architect-analyzes / CTO-decides division of roles.

[^11]: Ravikanth Konda — "Human-AI Collaboration in Software Teams: Evaluating Productivity, Quality, and Knowledge Transfer with Agentic and LLM-Based Tools," *International Journal of AI, BigData, Computational and Management Studies*, February 17, 2026. https://ijaibdcms.org/index.php/ijaibdcms/article/view/418
 Sprint AI-primary cap change decision requirements; capacity planning findings from quarterly reviews as annual planning inputs; resource allocation decisions at the CTO level.

[^12]: Kyros — "The Vibe Coding Crisis: How AI-Generated Technical Debt Is Costing Companies Millions," March 2026. https://usekyros.ai/blog/vibe-coding-crisis-ai-technical-debt
 Annual governance summary as an annual planning input; technical debt roadmap candidacy criteria; the early warning function of quarterly reviews in annual planning.

[^13]: Stack Overflow — "2025 Developer Survey," Stack Overflow, December 2025. https://survey.stackoverflow.co/2025/
 Technical debt patterns that surface in quarterly reviews; the relationship between quarterly governance findings and engineering roadmap prioritization.

[^15]: Dex Horthy (YC Root Access) — "Advanced Context Engineering for Agents," YouTube, August 2025. https://www.youtube.com/watch?v=IS_y40zY-hc
 - Metric tracking configuration: how to configure Claude Code session logging to produce the data needed for quarterly review metric charts without manual data collection
 - Governance effectiveness measurement: specific approaches for quantifying the review round count reduction from spec.md requirements and similar governance interventions
 - Annual governance model review: how to structure the meta-review that determines whether the governance model's overhead is calibrated to the team's AI maturity level

[^a]: [Metrics: Team Health Dashboard](../Metrics/02-health-dashboard.md) — The health dashboard is the primary data source for quarterly review; the review cadence and the dashboard metrics are designed together.

[^b]: [Documentation: Architecture Decision Records](../Documentation/01-architecture-decision-records.md) — Quarterly review includes an ADR currency check; are recent architectural decisions documented and are existing ADRs still accurate.

[^c]: [Governance: Compliance and Audit](06-compliance-and-audit.md) — Quarterly health review feeds into compliance documentation; the review output is the evidence record for audit requirements.

[^d]: [Issues: Velocity Governance](../Issues/08-velocity-governance.md) — Quarterly review is the strategic-level intervention point for the velocity-governance feedback loop; it is where the CTO evaluates whether the balance has shifted.
