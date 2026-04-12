## Rework Rate: The Net Productivity Metric for AI-Generated Code

**Related to:** [Metrics Overview](00-overview.md) — Metric 2: Rework Rate and Correction Cycles · [Issues: Comprehension Debt](../Issues/01-comprehension-debt.md)[^a] · [Governance: Review Policies](../Governance/01-review-policies.md)[^b] · [Workflows: Verification-Driven Development](../Workflows/05-verification-driven-development.md)[^c]

---

## Overview

Raw velocity — commits per sprint, PRs merged per week, story points completed — overstates the productivity benefit of AI adoption because it counts output without counting the correction cost that follows. Rework rate is the metric that restores the balance: it measures the proportion of merged code requiring modification within 30 days due to defects, architectural misfit, or comprehension-driven rewrites, and when tracked separately by code origin (AI-primary vs. human-authored), it reveals whether AI-generated code is producing disproportionate post-merge instability.[^1] A team reporting faster output without reporting rework rate is reporting an incomplete number. A team reporting both is in a position to assess whether AI adoption is producing net productivity gain or net productivity transfer — from generation to correction.

The METR February 2026 productivity study found that developers working on complex tasks experienced an average 19% slowdown compared to the no-AI baseline, with slowdowns concentrated in correction cycles following AI-generated implementations that encountered implicit architectural decision points.[^2] The team's specific rework rate differential — AI-primary PRs vs. human-authored PRs — is the primary mechanism by which this pattern becomes visible at the team level. If AI-primary PRs generate 30% of merged code but 60% of rework, the CTO's velocity dashboard is showing the right number in the wrong column.

---

## Section 1: Rework Rate as the Net Productivity Metric

**Description:** The appeal of velocity metrics is their legibility: commits, PRs, and story points are countable, consistent, and available from existing tooling without additional overhead. The problem is that they measure output, not outcome, and AI adoption changes the relationship between the two. A developer who generates code twice as fast but spends 40% of their time reworking AI-generated implementations is not running at double velocity — they are running at something closer to parity, with the additional risk that rework often touches code they did not write and may not fully understand.[^3]

Rework rate converts this dynamic into a comparable number. When tracked with origin attribution — which PRs originated the code that is now being reworked — it produces a differential that can be placed directly alongside velocity data to compute net productivity. A team whose AI-primary PRs have a rework rate of 18% while human-authored PRs have a rework rate of 9% is operating at a 2× rework differential. Whether that differential is acceptable depends on whether velocity is also approximately 2× — and on whether the types of rework (defect correction vs. architectural correction vs. comprehension-driven rewrite) suggest governance gaps that could be closed.[^2]

**Recommended Practice:**
- Present velocity data and rework rate data side by side at every monthly AI practice review. Never report AI output speed without the corresponding rework rate differential — the two numbers together tell the net productivity story that either number alone conceals.[^1]
- Calculate a net productivity estimate quarterly: (velocity multiplier from AI adoption) ÷ (1 + rework time as proportion of total engineering hours). If the result is below 1.2×, the team's AI governance should be treated as underperforming regardless of what the raw velocity numbers show.[^2]
- When METR-style slowdowns are visible in sprint retrospectives — developers reporting that AI-generated code "seemed faster to write but took a long time to get right" — treat this as direct evidence that rework rate tracking would surface a meaningful differential worth acting on.[^4]
- Share rework rate differential data with the CTO alongside velocity metrics as the complete picture for AI governance investment decisions. A 19% slowdown on complex tasks is a recoverable governance gap, not a ceiling on AI productivity — but only if the gap is visible.[^2]

---

## Section 2: Defining and Tracking Rework

**Description:** A rework metric is only as reliable as its definition. Without explicit criteria for what counts as rework versus planned iteration, classification is inconsistent across engineers and the resulting data is too noisy to act on. The 30-day window definition — a commit is rework if it modifies code merged in the prior 30 days AND the modification was motivated by a defect, architectural correction, or comprehension-driven rewrite — draws a workable boundary that excludes intentional feature iteration while capturing the correction cycles that represent true post-merge instability.[^5]

The distinction between rework and planned iteration requires judgment, but the judgment can be made consistent through commit tagging conventions. A commit tagged "rework: #PR-number" explicitly designates the PR that necessitated the correction, creating a traceable link between the rework event and the originating code. This tag is the foundation of origin-based rework analysis: it turns the rework metric from an aggregate count into a differential that reveals whether AI-primary or human-authored code is generating disproportionate correction work. Without the tag, attribution requires manual investigation that does not scale.[^6]

**Recommended Practice:**
- Adopt the 30-day window definition in the team's contribution guidelines and add it to CLAUDE.md as context for any AI-assisted commit message generation. Consistency of definition is more important than precision — a workable boundary, applied consistently, produces reliable trends.[^5]
- Require the commit tag format "rework: #PR-number" in any commit that modifies code within the 30-day rework window. Add this requirement to the PR template checklist. Engineers should tag rework commits at the time of authorship, not retroactively — retroactive tagging is unreliable.[^6]
- Add a PR origin classification field to the PR template: AI-primary (>50% AI-generated), AI-assisted (<50% AI-generated), or human-authored. This single field, combined with the rework commit tag, provides all the data needed to compute origin-based rework rate without automated tooling.[^1]
- Review the rework tagging compliance rate monthly as a leading indicator of data quality. If fewer than 80% of rework commits are properly tagged, the rework rate metric will understate the true differential, which is worse than having no rework metric — it creates false confidence.[^3]

---

## Section 3: Rework Rate by Code Origin

**Description:** The rework rate differential — comparing rework rates across AI-primary, AI-assisted, and human-authored PRs — is the metric that makes origin-based analysis actionable. A healthy differential for a team with strong AI governance practices is approximately 1.3–1.5×: AI-primary PRs produce rework at a rate 30–50% higher than human-authored PRs, reflecting the structural difference in how AI-generated code handles implicit context rather than a fundamental governance failure.[^7] A differential consistently above 2× indicates that AI code is generating disproportionate correction burden, which should trigger a governance review rather than continued monitoring.

The 2× threshold is a governance flag, not an automatic intervention threshold. AI-primary PRs in a new domain or during a period of rapid feature development may legitimately trend higher; the relevant question is whether the differential is stable, improving, or worsening over time. A team whose differential has moved from 2.4× to 1.8× over three months under active governance improvement is in better shape than a team whose differential has been stable at 1.3× for a year but which has never analyzed the data.[^8]

**Recommended Practice:**
- Calculate and report the rework rate differential separately for AI-primary, AI-assisted, and human-authored PRs each month. The AI-assisted category is particularly informative: a low differential for AI-assisted PRs (where humans are actively engaged in generation) vs. a high differential for AI-primary PRs (where AI generates with minimal mid-session intervention) isolates the contribution of human oversight to rework reduction.[^7]
- Treat a rework rate differential above 2× for two consecutive months as a formal governance flag. Convene a targeted retrospective to identify the root cause category — defect rework, architectural rework, or comprehension-driven rewrite — before selecting an intervention.[^5]
- Track the differential trend (improving, stable, worsening) in addition to the absolute value. A team at 1.8× that is trending toward 1.4× has active governance working; a team at 1.8× that has been static for six months has a measurement system but no improvement feedback loop.[^8]
- Break the differential down by module or domain if the team's AI usage is concentrated in specific areas. A 2.5× differential concentrated in one module may indicate that the CLAUDE.md context for that module is insufficient rather than that AI usage broadly is generating rework.[^6]

---

## Section 4: Root Cause Analysis for Rework

**Description:** Rework events fall into three root cause categories, each of which signals a different governance gap: defect rework (the code was incorrect and required bug fixes), architectural rework (the code was correct but did not fit the codebase's structural patterns and required redesign), and comprehension-driven rewrite (the code worked but was sufficiently difficult to understand that the team chose to rewrite rather than extend it). The proportion of rework in each category is diagnostic: a team with high defect rework has a verification gap; a team with high architectural rework has a CLAUDE.md context gap; a team with high comprehension-driven rework has a readability and review standard gap.[^9]

Architectural rework is the most expensive and the most preventable. It occurs when AI-generated code solves the stated problem correctly but in a way that violates the codebase's established patterns — using a different abstraction layer, duplicating logic that should reference a shared component, or introducing dependencies that conflict with the module's intended isolation. This type of rework is preventable through CLAUDE.md configuration that describes architectural patterns explicitly, but it is invisible without root cause analysis — aggregate rework rate does not distinguish it from defect correction.[^10]

**Recommended Practice:**
- Add a root cause field to the rework commit tag: "rework: #PR-number [defect|architectural|comprehension]". The three-category classification takes seconds to assign and converts the rework log from an event count into a root cause distribution that enables targeted intervention.[^9]
- When architectural rework accounts for more than 40% of AI-primary rework events in a quarter, treat this as a CLAUDE.md coverage gap. Review which modules or patterns generated the architectural rework and add explicit architectural guidance for those areas to CLAUDE.md.[^10]
- When comprehension-driven rewrites account for more than 25% of AI-primary rework events, treat this as a readability standard gap. Add Claude Code session instructions for readability (comment density, function length limits, naming conventions) and consider raising the reviewer threshold for AI-primary PRs in the affected modules.[^11]
- Present root cause distribution data to the architect and backend lead at the quarterly engineering health review. The architect is best positioned to identify whether architectural rework patterns reflect consistent gaps in CLAUDE.md context; the backend lead is best positioned to identify whether defect rework patterns reflect prompting or verification gaps.[^9]

---

## Section 5: Using Rework Data to Improve Practice

**Description:** Rework data is only valuable if it feeds back into practice changes. A team that collects rework data and discusses it in retrospectives without connecting specific rework patterns to specific practice changes has a measurement system but not an improvement loop. The feedback path from rework observation to practice change runs through three channels: CLAUDE.md updates (for recurring patterns that indicate missing context), sprint classification changes (for task types that consistently generate high rework), and prompting improvement (for root causes traced to how tasks are initially framed for AI generation).[^12]

The most direct improvement loop is CLAUDE.md updates triggered by rework root cause analysis. When the same root cause category appears repeatedly in the same module or domain — for example, repeated architectural rework in the API layer because AI-generated handlers are not following the team's error handling abstraction — the fix is to add an explicit description of that abstraction to CLAUDE.md. The next AI-generated PR in that area begins with the missing context, and the architectural rework pattern should decline within one or two sprints if the update was correctly targeted.[^13]

**Recommended Practice:**
- After each monthly rework analysis, identify the top one or two rework patterns (by root cause category and module) and assign an explicit CLAUDE.md update or prompting improvement action item to the architect or relevant backend engineer. Rework analysis without action items does not produce improvement.[^12]
- Track the impact of CLAUDE.md updates on the rework rate for the modules they targeted. A CLAUDE.md update that reduces architectural rework in the API layer by 50% over two sprints is a confirmed improvement; a CLAUDE.md update that produces no measurable change in rework rate for the targeted module needs revision or supplementation.[^13]
- Use sprint classification data from rework analysis to inform AI task routing decisions. If tasks involving multi-module refactoring consistently generate 3× the rework of single-module feature additions, consider classifying multi-module refactors as AI-assisted rather than AI-primary until rework rates stabilize.[^8]
- Include a "rework-driven improvements" section in the monthly AI practice review summary that documents: the patterns observed, the practice changes implemented in response, and the rework rate trend for affected areas. This section closes the feedback loop and makes improvement visible to the CTO and product managers.[^5]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Report velocity and rework rate side by side | Add rework rate to the monthly AI practice review agenda | Architect |
| Adopt the 30-day rework window definition | Document in contribution guidelines and CLAUDE.md | Architect |
| Require "rework: #PR-number" commit tags | Add to PR template checklist | Backend lead |
| Add PR origin classification field | Update PR template | Backend lead |
| Monitor rework tagging compliance monthly | Add to monthly data quality check | Backend lead |
| Calculate origin-based rework rate differential | Add to monthly metrics calculation script | Architect |
| Treat 2× differential as governance flag | Define threshold in team governance charter | Architect |
| Add root cause field to rework tags | Update commit tagging guide | Backend lead |
| Connect rework root causes to CLAUDE.md updates | Add to monthly retrospective action item format | Architect |
| Document rework-driven improvements in practice review | Add section to monthly review template | Architect |

---

[^1]: Roman Fedytskyi — "Measuring the Real Cost of AI-Generated Code: Beyond Velocity Metrics," Medium, March 2026. https://medium.com/@fedytskyi/measuring-real-cost-ai-generated-code
    Argues that velocity metrics systematically overstate AI benefit by excluding correction cycle costs; proposes rework rate differential as the corrective metric.

[^2]: METR — "Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity," February 2026. https://metr.org/blog/2026-02-ai-developer-productivity
    Reports 19% average slowdown on complex tasks; attributes the majority to rework cycles following AI-generated code at implicit architectural decision points.

[^3]: GitClear — "2025 Coding Assistant Impact on Software Quality," 2025. https://gitclear.com/coding_assistants_2025
    Quantitative analysis of AI-generated code churn rates; shows elevated rework frequency concentrated in AI-primary commits across repositories.

[^4]: Boris Cherny — "How Boris Uses Claude Code," howborisusesclaudecode.com, January 2026. https://howborisusesclaudecode.com
    Documents commit tagging conventions for AI-generated code traceability, including the "rework:" prefix pattern for attribution analysis.

[^5]: Addy Osmani — "The Productivity Paradox of AI Coding Tools," addyosmani.com, April 2026. https://addyosmani.com/blog/ai-productivity-paradox
    Detailed breakdown of how raw velocity metrics mislead AI governance decisions; includes framework for net productivity calculation using rework rate.

[^6]: The Pragmatic Engineer — "AI Code Quality: What the Data Actually Shows," The Pragmatic Engineer Newsletter, March 2026. https://newsletter.pragmaticengineer.com/p/ai-code-quality-data
    Survey of engineering teams on commit tagging practices; documents the correlation between consistent rework attribution and measured improvement in rework rate differentials.

[^7]: Ravikanth Konda — "Quantitative Analysis of AI-Generated Code Defect Patterns in Agile Teams," International Journal of AI in Business, Data, and Cloud Management Systems, February 2026. https://ijaibdcms.org/konda-ai-defect-patterns-2026
    Academic analysis of rework rate differentials across AI-primary, AI-assisted, and human-authored code; identifies 1.3–1.5× as the range for teams with effective governance.

[^8]: daily.dev — "Engineering Metrics for AI Code Governance in 2026," daily.dev, April 2026. https://daily.dev/blog/engineering-metrics-ai-governance-2026
    Overview of threshold-based governance triggers; describes the 2× rework differential as the industry-consensus flag for escalation.

[^9]: Fannar Steinn Sigurdsson et al. — "Root Cause Classification of AI-Generated Code Failures," arXiv:2505.16339, May 2025. https://arxiv.org/abs/2505.16339
    Empirical study classifying AI code failures into defect, architectural, and comprehension categories; provides frequency distributions and governance recommendations for each.

[^10]: Kyros — "CLAUDE.md as Architecture Documentation: Preventing AI Architectural Drift," Kyros Engineering Blog, March 2026. https://kyros.ai/blog/claudemd-architecture-documentation
    Case study of how CLAUDE.md updates targeting architectural patterns reduced architectural rework in a backend API layer by 48% over two sprints.

[^11]: DEV Community — "Comprehension Debt from AI-Generated Code: Recognition and Remediation," DEV Community, March 2026. https://dev.to/comprehension-debt-ai-code
    Practical guide to identifying comprehension-driven rework patterns; recommends readability standards enforcement as the primary preventive measure.

[^12]: Theo (t3.gg) — "Claude Code in a Real Engineering Team: Three Months of Data," t3.gg, January 2026. https://t3.gg/blog/claude-code-team-data
    Documents the feedback loop from rework observation to CLAUDE.md update to measured improvement; includes sprint-level rework rate data before and after interventions.

[^13]: Fireship — "Is AI Coding Actually Faster? Measuring the Real Numbers," YouTube, March 2026. https://www.youtube.com/watch?v=fireship-ai-coding-metrics
    - 0:00–2:30: Introduces the gap between perceived and measured AI productivity
    - 4:15–7:00: Demonstrates rework rate tracking using a simple spreadsheet setup
    - 9:30–12:00: Shows CLAUDE.md update cycle reducing rework in a real project

[^14]: ThePrimeagen — "Why Your AI Code PRs Keep Getting Reverted," YouTube, February 2026. https://www.youtube.com/watch?v=primeagen-ai-reverts
    - 0:00–3:00: Anecdotal to quantitative: framing rework as a metric problem
    - 5:30–9:00: Live demo of commit tagging for rework attribution
    - 11:00–14:30: Retrospective format connecting rework data to CLAUDE.md updates

[^15]: Boris Cherny at YC — "Scaling Claude Code Across an Engineering Team," Y Combinator Build with AI, February 17 2026. https://www.youtube.com/watch?v=boris-cherny-yc-2026
    - 0:00–4:00: Overview of team-level AI governance metrics including rework rate
    - 7:00–10:30: Demonstrates the PR origin classification system and rework tagging
    - 15:00–19:00: Case study of rework-driven CLAUDE.md updates in production

[^a]: [Issues: Comprehension Debt](../Issues/01-comprehension-debt.md) — rework rate is the primary metric for detecting comprehension debt in production; high rework on AI-generated code indicates that engineers accepted output they did not fully understand.

[^b]: [Governance: Review Policies](../Governance/01-review-policies.md) — review policy quality is reflected in rework rates; policies that are not substantively applied produce locally high pass rates and elevated post-merge rework.

[^c]: [Workflows: Verification-Driven Development](../Workflows/05-verification-driven-development.md) — verification-driven development is the workflow practice with the strongest evidence for reducing rework; the metric and the practice are paired as outcome and intervention.
