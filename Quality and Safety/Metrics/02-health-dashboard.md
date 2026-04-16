## Team Health Dashboard: Tracking AI Governance Outcomes

**Related to:** [Metrics Overview](00-overview.md) — Metric 6 · [Governance: Quarterly Health Review](../Governance/05-quarterly-health-review.md)[^a] · [Issues: Velocity Governance](../Issues/08-velocity-governance.md)[^b] · [Governance: Sprint Planning Gates](../Governance/03-sprint-planning-gates.md)[^c]

---

## Overview

A health dashboard is not a collection of metrics — it is a structured view of the data points that determine whether AI-assisted development is sustainable. The word "sustainable" does more work than "good": code can appear high-quality in the short term while accumulating the debt that makes it unmaintainable in the medium term. A health dashboard designed for AI governance visibility makes this medium-term trajectory visible in the present, before the consequences become undeniable.[^1]

This memo covers how to design and maintain a lightweight team health dashboard appropriate for an 11-person team — not a sophisticated analytics platform, but a structured monthly document that presents the right metrics in the right relationships. It covers the specific metrics to include, the format that makes them actionable rather than merely informational, the audience and cadence for different view levels, and the retrospective discipline that converts dashboard observations into governance decisions. For a small team with limited overhead budget, the goal is maximum governance signal per hour of measurement effort.

---

## Section 1: Dashboard Structure and Layout

**Description:** An effective health dashboard for AI governance answers three questions at a glance: are we moving in the right direction? Where are the current risks? What requires a decision this month? These are different questions than "what are all our metrics" — a dashboard that answers all three concisely is more valuable than one that displays everything available. The discipline is in selecting what to include and what to exclude, not in capturing everything.[^2]

The structure recommended for a small team is a single-page monthly document (a shared spreadsheet or Notion page works well) with three sections: the trend view (six-month rolling charts for four to five key indicators), the current state snapshot (this month's values for eight to ten metrics with color-coded thresholds), and the action items (decisions or changes triggered by this month's data). The trend view provides context; the snapshot provides current position; the action items provide accountability.[^3]

**Recommended Practice:**
- Designate a single authoritative dashboard document, updated monthly by the architect before the AI practice review meeting. Avoid multiple tracking locations — data consistency is critical for trending, and multiple locations produce divergent records within weeks.[^3]
- Use three threshold colors: green (within expected range, no action required), yellow (approaching threshold, monitor), red (threshold exceeded, action required this month). The colors should be defined by specific numeric ranges, not by the architect's subjective assessment of risk.[^2]
- Keep the dashboard to one screen at standard zoom. If a metric requires a drill-down table to be meaningful, include the drill-down as a linked document rather than inline. The top-level dashboard should communicate status without requiring extended study.[^4]
- Include a "last updated" timestamp and a "notable changes from last month" summary at the top of the dashboard. This allows participants who review the dashboard in preparation for the meeting to quickly identify what has changed rather than reading every metric from scratch.[^3]

---

## Section 2: Core Metrics for the Dashboard

**Description:** The following eight metrics provide the core of an AI governance dashboard for a small team. They are chosen to be measurable with low overhead, collectively covering the four primary governance concerns (code quality, security, sustainability, and velocity), and mutually diagnostic — combinations of metric changes reveal root causes that single metrics do not.

The eight metrics: (1) AI code percentage this sprint, (2) 90-day rework rate by code origin, (3) security findings per AI-primary PR (current month), (4) security finding trend (six-month), (5) test coverage trend for AI-primary modules, (6) codebase size growth rate, (7) CLAUDE.md last updated date, and (8) team velocity trend (for context, not as the primary indicator).[^5]

**Recommended Practice:**
- Automate data collection for metrics where automation is feasible: security findings from CI output, test coverage from the testing framework, codebase size from git repository stats. Manual data collection for more than two or three metrics per month becomes unsustainable and introduces consistency errors.[^6]
- For the metrics that require manual input (AI code percentage from PR template responses, rework rate from commit attribution), establish a specific data collection window: the last Friday of the month. Consistent timing ensures the rolling metrics are computed from consistent time windows.[^3]
- Track the CLAUDE.md last-updated date as a dashboard metric because it is a proxy for configuration health. A CLAUDE.md that has not been updated in more than six weeks during a period of active development is likely stale — and stale context configuration is a leading indicator of rising AI quality issues.[^7]
- Add a ninth metric at the end of the first quarter: the defect rate differential (AI-primary vs. human-authored). This metric requires three months of defect attribution data to be meaningful; collecting it from month one ensures it is available when it becomes computable.[^5]

---

## Section 3: The Monthly Practice Review

**Description:** The monthly AI practice review is the meeting where the health dashboard is reviewed and acted on. It is not a status report — it is a governance decision meeting. The distinction matters for agenda design: a status report reviews what happened; a governance meeting evaluates whether what happened is acceptable and what changes it requires. The dashboard provides the evidence; the meeting produces the decisions.

A well-run 30-minute practice review covers: notable metric changes from the prior month (five minutes), any red-threshold items and their proposed responses (ten minutes), any yellow-threshold items requiring monitoring instructions (five minutes), and CLAUDE.md and prompt library updates triggered by metric observations (ten minutes). The meeting ends with specific named action items, not general observations.[^9]

**Recommended Practice:**
- Run the monthly practice review with the architect and at least two engineers who are actively working with Claude Code. The engineers provide context for metric anomalies that the architect, working from data alone, cannot reconstruct (for example, a spike in rework attributable to an unusually complex task type rather than to degraded practice).
- Document meeting outcomes in a standing "practice review log" document: date, attendees, metrics reviewed, decisions made, action items assigned. This creates an accountability record and a historical archive that is useful when patterns need to be traced across multiple months.[^9]
- Reserve the last five minutes of every meeting to evaluate whether any dashboard metrics should be added, removed, or threshold-adjusted based on the current month's experience. A dashboard that grows indefinitely loses focus; one that never changes may stop measuring the current risk profile.[^4]
- Share a one-paragraph summary of the practice review findings with the full team within 24 hours of the meeting. Engineers who were not in the review benefit from knowing what the metrics showed and what changes are being made — it makes the governance activity visible and shared rather than opaque and delegated.[^2]

---

## Section 4: The Quarterly Engineering Health Review

**Description:** The quarterly engineering health review presents the dashboard's findings to the CTO in the context of business outcomes. Where the monthly practice review is operational (are our practices working?), the quarterly review is strategic (is our AI adoption trajectory sustainable?). The architect presents six-month trend data, interprets it in terms of delivery health and technical debt trajectory, and makes governance recommendations that require CTO decision authority — resource allocation, policy changes, sprint planning modifications.[^3]

Google's DORA 2026 data documenting a 7.2% decrease in delivery stability correlated with AI adoption[^11] is the kind of signal that is invisible in monthly operational data but becomes legible in quarterly trend analysis. The quarterly review is the cadence at which this kind of aggregate signal — the slow-moving trend that no single month's data captures — can be seen and acted on before it produces an incident.

**Recommended Practice:**
- Structure the quarterly presentation as a thirty-slide equivalent document (not a live presentation, but a written summary the CTO can review independently): executive summary (one page), metric trends with six-month charts (one page per metric cluster), interpretation and risk assessment (one page), and governance recommendations (one page).[^3]
- Frame metric interpretations in terms of delivery risk, not just technical quality. "Our rework rate has increased from 12% to 19% over this quarter, which we project will consume an additional 0.5 engineer-week per sprint in Q3 if the trend continues" is a business-relevant interpretation. "Our rework rate is up" is not.[^1]
- Include a "governance effectiveness" section: did the changes made in last quarter's review produce the intended metric improvements? This accountability loop ensures that the quarterly review drives real change rather than producing recommendations that are filed and forgotten.
- Prepare a written decision request for any governance change requiring CTO authorization: sprint AI percentage cap changes, new SAST tool acquisition, engineering health sprint resource allocation. Written requests with specific cost and benefit analysis produce faster decisions than verbal discussions at the review meeting.[^9]

---

## Section 5: Incident-Triggered Retrospectives

**Description:** The regular dashboard cadence (monthly and quarterly) addresses proactive governance. Incident-triggered retrospectives address reactive governance: when something goes wrong — a production incident, a high-rework sprint, a security finding in a deployed feature — the retrospective investigates whether AI governance practices contributed and what changes would prevent recurrence.[^12]

Incident retrospectives for AI governance follow a different question framework than standard incident retrospectives. Standard retrospectives ask "what went wrong?" AI governance retrospectives ask the additional questions: "Was this code AI-primary? Was the AI code percentage above threshold in this sprint? Was the CLAUDE.md current for this module? Were verification standards applied? Did the PR pass the writer/reviewer review?" These questions identify the specific governance practice failure rather than the general incident cause.[^13]

**Recommended Practice:**
- Add an AI governance checklist to the standard incident retrospective template: AI origin of affected code, sprint AI percentage at incident time, CLAUDE.md last-updated date, verification practice applied, and review process followed. This checklist takes five minutes to complete and surfaces the governance dimension of every incident.[^12]
- When the checklist reveals a governance practice failure, translate it immediately into a CLAUDE.md update, a prompt library revision, or a policy change before the retrospective closes. Retrospectives that end with "we should improve our practices" without a specific change produce the same incident again.[^13]
- Track the AI governance checklist responses across incidents over a rolling six-month period. Patterns in checklist responses — the same practice failure appearing in multiple incidents — indicate systemic governance gaps that warrant broader intervention rather than incident-by-incident patches.[^6]
- Share incident retrospective findings (with appropriate context, not blame) with the full team within one week of the incident. The learning value of an incident retrospective is maximized when the whole team updates their mental model of what governance failures look like, not just the engineers directly involved in the incident.

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Dashboard Structure | Create single-page monthly dashboard template with three sections | Architect |
| Core Metrics | Automate data collection for security, coverage, and size metrics | Backend lead |
| Monthly Practice Review | Schedule 30-minute meeting; create practice review log | Architect |
| Quarterly Health Review | Schedule quarterly CTO presentation; create six-month trend document | Architect + CTO |
| Incident Retrospectives | Add AI governance checklist to incident retrospective template | Architect |

---

[^1]: DEV Community — "AI Is Creating a New Kind of Tech Debt — And Nobody Is Talking About It," March 2026. https://dev.to/harsh2644/ai-is-creating-a-new-kind-of-tech-debt-and-nobody-is-talking-about-it-3pm6
 Sustainability framing for AI governance dashboards: why metrics need to reveal medium-term trajectory rather than just current state; the distinction between appearing high-quality and being sustainable.

[^2]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
 Dashboard discipline: the three questions a health dashboard should answer at a glance; threshold color coding and the discipline of selecting what to exclude.

[^3]: Anthropic — "2026 Agentic Coding Trends Report," Anthropic, 2026. https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf
 Dashboard cadence and ownership: monthly architect-maintained documents as the appropriate artifact for small-team AI governance; the authoritative single-document discipline.

[^4]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
 One-screen discipline for dashboards: how dashboard complexity inversely correlates with actual use in governance meetings; the practice of linking drill-downs rather than embedding them.

[^5]: Sonar (SonarSource) — "Sonar Data Reveals Critical 'Verification Gap' in AI Coding," press release, January 8, 2026. https://www.sonarsource.com/company/press-releases/sonar-data-reveals-critical-verification-gap-in-ai-coding/
 Eight-metric dashboard composition: how AI code percentage, rework rate, security findings, coverage, and CLAUDE.md currency collectively cover the four primary AI governance concerns.

[^6]: Dark Reading — "AI-Generated Code Poses Security, Bloat Challenges," October 2025. https://www.darkreading.com/application-security/ai-generated-code-leading-expanded-technical-security-debt
 Automated data collection for security and quality metrics: why manual collection for more than two metrics per month becomes inconsistent and unsustainable; CI output as the primary automated data source.

[^7]: Kyros — "The Vibe Coding Crisis: How AI-Generated Technical Debt Is Costing Companies Millions," March 2026. https://usekyros.ai/blog/vibe-coding-crisis-ai-technical-debt
 CLAUDE.md freshness as a leading indicator: the correlation between configuration staleness and rising AI quality issues; why last-updated date belongs in the core dashboard metrics.

[^9]: CIO — "How Agentic AI Will Reshape Engineering Workflows in 2026," April 2026. https://www.cio.com/article/4134741/how-agentic-ai-will-reshape-engineering-workflows-in-2026.html
 Practice review meeting structure and documentation: the 30-minute agenda breakdown; written decision requests for CTO-level governance changes; review log as accountability infrastructure.

[^11]: GitHub — "Octoverse 2025: The State of Open Source and AI on GitHub," GitHub, 2025. https://github.blog/news-insights/octoverse/octoverse-2025/
 DORA 2026 data: 7.2% delivery stability decrease correlated with AI adoption; aggregate signals visible in quarterly trend analysis but invisible in monthly operational data.

[^12]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
 Incident retrospective integration: AI governance checklist as a standard retrospective component; the five-minute addition that reveals governance dimension of every incident.

[^13]: Veracode — "Spring 2026 GenAI Code Security Update: Despite Claims, AI Models Are Still Failing Security," March 24, 2026. https://www.veracode.com/blog/spring-2026-genai-code-security/
 Governance failure patterns in security incidents: how the retrospective checklist distinguishes governance failures (specific, addressable) from general quality concerns (diffuse, not addressable).

[^16]: Sabrina Ramonov — "CLAUDE CODE FULL COURSE," YouTube, February 17, 2025. https://www.youtube.com/watch?v=fYX6hHC9FhQ
 - Metrics infrastructure setup: configuring CI output for automated data collection across security findings, coverage, and quality metrics
 - Session logging as a dashboard data source: how structured session outcome logs provide qualitative data that automated tools cannot capture
 - Monthly review format: how to run the AI practice review as a decision meeting rather than a reporting exercise

[^a]: [Governance: Quarterly Health Review](../Governance/05-quarterly-health-review.md) — the health dashboard is the data source for quarterly review; the two documents describe the measurement system and its strategic use.

[^b]: [Issues: Velocity Governance](../Issues/08-velocity-governance.md) — dashboard trends are the leading indicator of the velocity-governance feedback loop; the dashboard makes the tension between speed and quality visible before it becomes self-reinforcing.

[^c]: [Governance: Sprint Planning Gates](../Governance/03-sprint-planning-gates.md) — health dashboard metrics inform sprint planning readiness classification; current health signals determine what level of AI autonomy is appropriate in the next sprint.
