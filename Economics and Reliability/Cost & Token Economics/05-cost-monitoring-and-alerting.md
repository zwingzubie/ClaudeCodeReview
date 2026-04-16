## Cost Monitoring and Alerting

**Related to:** [Cost & Token Economics Overview](00-overview.md) — Cost Area 5 · [Metrics: Health Dashboard](../Metrics/02-health-dashboard.md) · [Governance: Quarterly Health Review](../Governance/05-quarterly-health-review.md) · [Tooling: Hooks and Automation](../Tooling & Configuration/02-hooks-and-automation.md)

---

## Overview

Cost monitoring converts spend data into decisions. A team with a budget but no monitoring discovers overruns only at invoice time; a team with monitoring discovers them when there is still time to act. The monitoring infrastructure for Claude Code spend is available through the Anthropic API usage console, supplemented by hook-driven session logging that provides cost attribution at the session level. The alerting layer — automated thresholds that notify the architect when spend is trending above plan — transforms monitoring from a passive dashboard into an active governance mechanism.[^1]

The monitoring requirements for a team of 11 are modest. The primary artifacts are a daily spend trend visible to the architect, a per-engineer usage breakdown updated at least weekly, and two automated alerts: a planning alert at 70% of monthly budget and a restrict alert at 90%. These are not technically complex to configure — the Anthropic console provides the raw data, and the alerting can be implemented through simple integrations with the team's existing notification infrastructure (Slack, email, or a shared dashboard). The effort required to configure this monitoring is measured in hours; the budget visibility it provides persists for the life of the team's AI adoption.[^2]

---

## Section 1: Anthropic Console Usage Dashboard

**Description:** The Anthropic console's usage dashboard is the primary monitoring surface for team API spend. It provides total token usage by model, cost breakdown by date range, and — when API keys are organized by engineer — per-key usage attribution. For a team that has configured per-engineer API keys (see Budget Governance), the console provides the raw data for both team-level and per-engineer cost tracking without requiring any additional tooling.[^3]

The limitation of the console dashboard is that it provides aggregated token and cost data, not session-level attribution. It shows that $42 was spent on Sonnet input tokens on a given day, but not which sessions generated that spend or which tasks those sessions addressed. Session-level attribution requires supplementary logging — either through Claude Code hooks or through a manual session log maintained by engineers.[^1]

**Recommended Practice:**
- Designate a shared Anthropic organization account for the team, with per-engineer API keys issued from that account. This configuration makes all usage visible at the organization level in the console while allowing per-engineer attribution through key-level filtering.[^3]
- Review the console usage dashboard on a fixed weekly cadence — 10 minutes every Monday morning is sufficient for early trend detection. The weekly review should cover: current week spend vs. weekly budget target, model distribution vs. team norms (Sonnet as default, Opus for justified escalation), and any day with spend materially above the daily average.[^2]
- Export usage data monthly to the team's shared cost tracking document. Monthly exports enable trend analysis that the console's default views do not surface directly: month-over-month spend trend, model mix trend over time, and correlation between sprint AI intensity and cost outcomes.[^1]
- Share the console dashboard access with the CTO for the quarterly engineering health review. Direct dashboard access gives the CTO the ability to interrogate spend data independently rather than relying solely on the architect's summary — an important accountability layer for significant infrastructure spend.

---

## Section 2: Hook-Driven Session Cost Logging

**Description:** Claude Code hooks can execute arbitrary shell commands in response to session lifecycle events, including session completion. A Stop event hook that logs token usage, model tier, and session metadata to a structured log file provides session-level cost attribution that the console dashboard cannot. This log becomes the data source for cost analysis at the session and task-type level — the granularity required to identify which session patterns are driving disproportionate spend.[^4]

Hook-based session logging requires minimal setup: a Stop event hook script that reads session metadata from the Claude Code environment and appends a structured log entry to a shared file. The log entry should capture: timestamp, engineer identifier, model tier used, approximate input tokens, approximate output tokens, and a brief session description (either the engineer's session brief or the first prompt message, truncated). This data, accumulated over a month, provides enough resolution to identify cost outliers and calibrate model selection norms.[^5]

**Recommended Practice:**
- Implement a Stop event hook that logs session cost metadata to a shared CSV or JSON log file. Minimal required fields: timestamp, API key identifier (engineer attribution), model used, input tokens, output tokens, and first 200 characters of the session description. The hook runs automatically at session end and requires no engineer action after initial setup.[^4]
- Review the session log weekly for sessions above the per-session cost threshold defined in Budget Governance ($5 or the team's chosen threshold). For each flagged session, confirm that it was pre-notified per the approval workflow and that the session description aligns with a justified high-cost task type.[^5]
- Aggregate the session log monthly by task type to identify cost distribution patterns: which task types drive the most spend, which drive the most Opus usage, and which have the highest cost-per-turn ratios. These aggregates inform both model selection policy refinement and CLAUDE.md context optimization priorities.[^2]
- Make the session log available to all engineers, not just the architect. Transparency about session-level costs across the team builds shared cost intuition more effectively than top-down policy enforcement. Engineers who can see that a colleague's session cost $0.40 for the same task type they spent $4.00 on have a concrete prompt to investigate their own session patterns.

---

## Section 3: Automated Alerting Configuration

**Description:** Manual dashboard review catches cost trends after they have persisted for a week or more. Automated alerts catch them in near-real-time, enabling corrective action before a brief overrun becomes a significant overage. The two critical alert thresholds for a team of 11 are a planning alert (70% of monthly budget consumed) and a restrict alert (90% of monthly budget consumed). The planning alert triggers a mid-month review; the restrict alert triggers a hold on non-critical high-cost sessions until the architect has reviewed current usage and confirmed the remaining budget is sufficient to complete the sprint.[^2]

The alerting infrastructure can be implemented through the Anthropic console's usage alert configuration, through third-party cost monitoring tools that integrate with the Anthropic API, or through a simple daily usage check script that compares cumulative spend to the monthly budget and posts to a Slack channel when thresholds are crossed. The implementation choice should be the simplest one that is reliably maintained — an automated Slack alert that fires every morning is more effective than a sophisticated dashboard that requires manual checking to produce value.[^1]

**Recommended Practice:**
- Configure Anthropic console usage alerts at 70% and 90% of the monthly budget. These alerts notify via email and can be configured to notify multiple recipients (architect and CTO). The console alert is the minimum viable alerting implementation and requires no additional tooling.[^3]
- For teams already using Slack as a primary communication channel, supplement console email alerts with a Slack integration: a daily cron job that posts current month spend vs. budget to a `#engineering-costs` channel. Daily visibility in Slack is more likely to generate action than an email alert that may not be read promptly.[^2]
- Define the response procedure for each alert threshold before it fires. The planning alert response: architect reviews current month usage, identifies whether any scheduled high-cost sessions should be deferred, confirms remaining budget is sufficient for sprint completion. The restrict alert response: non-critical Opus and agentic sessions are held pending architect review; incident-response sessions proceed with post-incident review.[^2]
- Treat two consecutive planning alerts (70% of budget consumed before mid-month in two successive months) as a signal to increase the budget baseline rather than continuing to restrict usage. Chronic underbudgeting creates governance friction that reduces AI utilization below the optimal level — the goal of budget governance is sustainable utilization at planned cost, not maximum restriction.

---

## Section 4: Quarterly Cost Review Reporting

**Description:** The quarterly cost review is the governance artifact that connects monthly monitoring data to budget decisions, policy updates, and CTO visibility. It converts the granular data from the Anthropic console and session logs into a four-quarter trend that shows whether the team's AI spend is stable, growing, or declining relative to output value. This trend is the CTO's primary data point for evaluating whether the AI investment is performing as expected and whether the team's governance practices are keeping pace with usage growth.[^6]

The quarterly review does not require extensive data analysis — a well-maintained monthly cost log and session summary provide sufficient data for a 10-slide review. The review's value is not in analytical sophistication but in the discipline of regular structured reflection on spend trends, model selection patterns, and the governance practices that produced them.[^1]

**Recommended Practice:**
- Prepare a quarterly cost report covering: total API spend by month for the quarter, model mix distribution (% Haiku / Sonnet / Opus) and its trend, cost-per-feature-delivered, top-5 highest-cost sessions and their outcomes, and one governance recommendation for the next quarter (a model selection policy update, a CLAUDE.md optimization, or a monitoring improvement).[^6]
- Present the quarterly cost report to the CTO in the existing quarterly engineering health review (see Governance: Quarterly Health Review). A 10-minute standing segment on AI infrastructure costs positions AI spend as managed infrastructure, not as uncontrolled discretionary spend.[^2]
- Compare quarterly spend to the prior quarter's forecast: was the budget allocation accurate? If actual spend was consistently below allocation, reduce the allocation and redirect budget to other engineering infrastructure. If consistently above, increase the allocation and identify which governance practices failed to contain spend at plan.[^1]
- Include a cost efficiency trend in the quarterly report: cost-per-session over the four quarters. A declining cost-per-session trend indicates that context optimization, model selection calibration, and prompt caching are improving over time — the team is getting more efficient at using AI, not just spending more. A flat or rising trend despite stable output quality indicates governance practices are not improving and warrants a dedicated optimization sprint.

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Console Dashboard | Configure per-engineer API keys; establish weekly review cadence | Architect |
| Hook-Based Session Logging | Implement Stop event hook for session cost logging | Backend lead |
| Automated Alerting | Configure 70%/90% console alerts; add daily Slack spend post | Architect |
| Quarterly Cost Review | Prepare quarterly report template; add to engineering health review agenda | Architect + CTO |

---

[^1]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    API key organization for spend attribution; session logging disciplines; budget governance as a continuous monitoring practice rather than a monthly accounting exercise.

[^2]: Laura Tacho (DX) — "How Are Engineering Leaders Approaching 2026 AI Tooling Budgets?" DX Blog, 2026. https://getdx.com/blog/how-are-engineering-leaders-approaching-2026-ai-tooling-budget/
    Alert threshold configuration for team-level budget governance; planning vs. restrict alert response procedures; quarterly cost review reporting format; Slack integration for daily spend visibility.

[^3]: Anthropic — "Models Overview," Anthropic API Documentation, 2026. https://docs.anthropic.com/en/docs/about-claude/models/overview
    Anthropic console usage dashboard features; API key organization for per-engineer attribution; usage export formats for monthly trend analysis.

[^4]: Anthropic — "Claude Code Hooks Reference," Claude Code Documentation, 2026. https://code.claude.com/docs/en/hooks
    Stop event hook configuration for session completion logging; session metadata available to hook scripts; hook output formats and log file integration patterns.

[^5]: Nir Gazit (Traceloop) — "From Bills to Budgets: How to Track LLM Token Usage and Cost Per User," Traceloop Blog, 2025. https://www.traceloop.com/blog/from-bills-to-budgets-how-to-track-llm-token-usage-and-cost-per-user
    Hook-driven session cost log schema; weekly session log review for per-session cost outliers; monthly aggregate analysis by task type; transparency as a team cost calibration mechanism.

[^6]: CIO — "How Agentic AI Will Reshape Engineering Workflows in 2026," April 2026. https://www.cio.com/article/4134741/how-agentic-ai-will-reshape-engineering-workflows-in-2026.html
    Quarterly cost review as governance infrastructure; cost-per-feature-delivered as the CTO-facing metric; AI spend reporting within the quarterly engineering health review structure.
