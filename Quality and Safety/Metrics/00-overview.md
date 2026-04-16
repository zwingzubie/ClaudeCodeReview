## Overview

For a team of 11 — 4 backend engineers, 3 frontend engineers, 1 architect, 1 QA engineer, and 2 product managers — using Claude Code at scale, the standard engineering metrics are no longer sufficient. Lines of code per sprint and feature velocity describe what AI makes easy to measure: raw output. They do not measure what AI makes easy to obscure: rework rates, comprehension coverage, architectural consistency, and security vulnerability accumulation. A team that optimizes exclusively for velocity metrics while AI silently accumulates technical debt is not running better — it is running faster toward the same problems described in the Issues section, with less visibility into when they will surface.[^1]

The teams that manage AI adoption successfully in 2026 are those that have added a second measurement layer to their existing engineering metrics — one that tracks the health of AI-generated output, not just its volume. Google's DORA 2026 research found a 154% increase in average PR size and a 91% increase in review time correlated with high AI adoption, alongside a 7.2% decrease in delivery stability.[^2] These are indicators that appear in retrospect; catching them in near-real-time requires deliberate tracking. Kyros's analysis found that rework rates increase 30–60% within six months of heavy AI adoption — but only become visible on dashboards if someone is tracking rework explicitly.[^3]

Six metric areas are documented below. They are designed to complement existing velocity metrics rather than replace them, providing the visibility required to govern AI adoption rather than just benefit from it.

---

## Metric 1: AI Code Volume and Composition

**Description:** The foundational metric for AI governance is knowing how much of the codebase is AI-generated and how that proportion is changing over time. Without this baseline, every other governance metric lacks context. A security vulnerability rate of 5% reads differently depending on whether it comes from 10% AI-generated code or 60% AI-generated code. A rework rate of 20% is a different signal if AI code comprises 50% of merged PRs versus 10%.[^4]

Sonar's January 2026 survey found that 42% of all committed code now originates from AI across surveyed development teams.[^4] Tracking this proportion for the team specifically — by PR, by engineer, and by module — creates the denominator against which all other AI quality metrics are evaluated. It also surfaces the threshold identified in the task decomposition workflow research: above 40% AI code per sprint, rework rates spike materially, making the 40% figure a meaningful monitoring threshold.[^5]

**Proposed Solution:**
- Add AI code percentage as a tracked field in the PR template: engineers mark whether a PR is AI-assisted, AI-primary, or human-authored. This self-reporting baseline can be supplemented with commit analysis tools as the practice matures.[^4]
- Track AI code percentage by module, not just total — some modules warrant higher scrutiny than others (authentication, payment processing, data access), and knowing that 80% of your authentication module is AI-generated is more actionable than knowing the overall average is 40%.[^5]
- Report the trend monthly rather than a point-in-time snapshot. A rising AI code percentage combined with declining test coverage or rising vulnerability counts is a compound signal that warrants intervention. A rising percentage with stable quality metrics suggests the team's governance practices are keeping pace.[^3]
- Establish 40% as a leading-indicator threshold for the sprint: if AI-primary PRs are trending to exceed 40% of sprint output, flag it in the weekly engineering standup before the sprint closes rather than discovering it in retrospect.[^5]

---

## Metric 2: Rework Rate and Correction Cycles

**Description:** Rework rate — the proportion of merged code that requires modification within 30 days due to defects, architectural misfit, or comprehension-driven rewrites — is the most direct indicator of whether AI governance is keeping pace with AI output velocity. A stable or declining rework rate suggests that verification practices, review standards, and CLAUDE.md configuration are doing their job. A rising rework rate, especially when velocity is also rising, indicates that AI output quality is degrading faster than governance is catching it.[^3]

GitClear's analysis of GitHub commit patterns from 2022 to 2025 found that as AI code generation increased, refactoring activity collapsed from 25% of commits to under 10% — meaning engineers were adding code without cleaning up the code that preceded it. This refactoring collapse is a leading indicator of rework accumulation: code that was never refactored becomes harder to maintain and more prone to defects as it evolves.[^6]

**Proposed Solution:**
- Define rework explicitly in the team's tracking system: a commit is rework if it modifies code merged in the prior 30 days to fix a defect, revert an architectural choice, or rewrite for comprehension reasons. Distinguish rework from planned iteration — not all post-merge changes are rework.[^3]
- Track rework rate by PR type (AI-primary vs. human-authored) and by module. If rework is concentrated in AI-primary PRs in a specific module, that is a signal that the CLAUDE.md context for that module is insufficient or that verification standards are not being applied there.[^6]
- Present rework rate to the CTO monthly alongside velocity metrics. A CTO who sees only velocity metrics has an incomplete picture of delivery health; the rework rate makes the quality dimension visible in a form relevant to leadership review.[^1]
- Set an alert threshold at 20% rework rate for AI-primary PRs per sprint. This is the level identified in the Anthropic Agentic Trends research at which rework costs begin to exceed the velocity gains from AI generation.[^5]

---

## Metric 3: Security Vulnerability Trends

**Description:** Security vulnerability rates in AI-generated code are not a static risk — they change as AI models change, as the team's CLAUDE.md context evolves, and as verification practices improve or degrade. Veracode's Spring 2026 analysis found that security pass rates for AI-generated code remain at 55% — meaning 45% of AI-generated code still introduces a known security flaw — and that this rate has not improved despite significant advances in model capability.[^8] Tracking the team's own vulnerability rate over time determines whether the team's specific practices are producing better outcomes than the industry baseline.

Automated scanning in CI/CD (per Tooling & Configuration — Hooks and CI/CD Integration) generates the raw data for this metric; the metrics practice is about trending, alerting, and acting on that data rather than just collecting it. A single-sprint vulnerability rate is noise; a three-month trend is a signal. A rising trend with unchanged AI adoption rates indicates deteriorating practice quality; a rising trend correlated with rising AI adoption rates indicates that governance is not keeping pace with output volume.[^8]

**Proposed Solution:**
- Run SAST scanning on every PR and record findings in a central tracking location (not just PR comments, which expire from active attention). Track severity distribution over time: critical, high, medium, and low findings per sprint, by PR type.[^8]
- Report the vulnerability trend in the monthly engineering health review alongside rework rate and AI code percentage. Veracode's finding that vulnerability rates are unchanged despite model improvements is a direct argument for not assuming that model upgrades will solve the security problem — practice changes are required.[^8]
- Track vulnerabilities by category (injection, authentication, secrets management, API exposure) as well as by total count. A team whose injection vulnerability count is declining but hardcoded credential findings are rising has a specific practice gap to address, not a general quality problem.[^9]
- Use the security vulnerability trend as input for CLAUDE.md updates: a recurring vulnerability type is a candidate for a CLAUDE.md prohibition. Adding "Never hardcode API keys or credentials" to CLAUDE.md is more reliable than continuing to catch the pattern in CI review.[^10]

---

## Metric 4: Session and Context Efficiency

**Description:** Claude Code session efficiency — the ratio of useful output to correction cycles and context resets within a session — is a metric that individual engineers rarely track but that provides valuable signal about team-level prompting and context health. A team where engineers consistently require four or more correction cycles per session is either prompting poorly, providing insufficient context, or working outside the task types where AI is effective. Identifying and addressing the root cause is more valuable than accepting correction cycles as a constant overhead.

Session efficiency also tracks whether context management practices (CLAUDE.md freshness, spec.md usage, /compact discipline) are functioning as intended. If session quality noticeably degrades in the second half of long sessions, context window saturation is a candidate cause — and the team's /compact and session-reset disciplines are the intervention.[^10]

**Proposed Solution:**
- Ask engineers to briefly log session outcomes in a shared document after non-trivial AI sessions: task type, number of correction cycles, whether the output met the standard on first pass, and any notable failures. This qualitative data is not available in any automated tracking tool and surfaces patterns invisible to code metrics alone.
- Aggregate session logs monthly to identify which task types most frequently require multiple correction cycles. These are candidates for prompt library improvement or for reclassification as tasks better suited to human-first execution.[^5]
- Track CLAUDE.md update frequency as a proxy for context health: a CLAUDE.md that has not been updated in over a month may be stale. If session correction cycles are rising and CLAUDE.md has not been updated, the combination is a signal that context has drifted from the codebase's current state.[^10]
- Use unusually high correction cycle rates on specific tasks as a trigger for a prompt retrospective: convene a brief session to identify whether the prompt structure, context injection, or verification criteria can be improved before the next similar task.[^12]

---

## Metric 5: Codebase Health Indicators

**Description:** AI-assisted development changes codebase health in ways that standard code quality tools do not fully capture. GitClear's longitudinal analysis found that AI adoption correlated with copy-paste code rising from 8.3% to 12.3% of all code and refactoring collapsing from 25% to under 10% of commits — structural changes that compound over time into significantly higher maintenance costs.[^6] These trends are invisible to velocity dashboards and require deliberate measurement to surface before they become critical.

A 2026 analysis of 304,362 AI-authored commits found that 24.2% of AI-introduced code quality issues survive to the latest revision of the repository — meaning they are not being caught in review or cleaned up after the fact.[^13] On a small team, this survival rate translates into a compounding debt: problems introduced by AI sessions accumulate in the codebase at a rate proportional to AI adoption, unless active cleanup practices offset them.

**Proposed Solution:**
- Track four codebase health indicators monthly: lines of code trend (growth rate), duplicate code percentage, test coverage trend, and cyclomatic complexity trend. Present these alongside velocity at the monthly engineering health review — together, they describe whether the codebase is becoming easier or harder to work in.[^6]
- Establish a "deprecation sprint" cadence: one sprint per quarter where a defined portion of engineering capacity is allocated to removing dead code, consolidating duplicates, and addressing the highest-complexity modules. This sprint should appear on the product roadmap as a first-class item.[^3]
- Use complexity trend as an early indicator of architectural drift. A module whose complexity is rising faster than its feature count suggests AI is generating locally coherent but globally inconsistent code in that area — a signal to check whether CLAUDE.md context is sufficient for that module.[^6]
- Report the age distribution of code by author type (AI-primary vs. human-authored) to identify the proportion of the codebase that will require comprehension-building before it can be safely modified. Modules with high AI authorship and no named human expert are the highest-risk candidates for the comprehension debt described in the Issues section.[^13]

---

## Metric 6: Retrospective Cadence and Learning

**Description:** Metrics are only as valuable as the decisions they inform. A team that collects AI governance metrics but does not review them in a structured retrospective context is collecting data for its own sake. The retrospective cadence — when data is reviewed, by whom, in what format, and what decisions it drives — determines whether metrics produce governance or just records.[^14]

The Anthropic 2026 Agentic Coding Trends Report found that organizations achieving the most consistent improvement in AI output quality shared a common characteristic: structured, regular retrospectives specifically focused on AI workflow quality rather than subsuming AI discussion into general sprint retrospectives.[^5] General retrospectives allocate time proportional to urgency; AI quality concerns are rarely urgent until they compound into an incident. A dedicated AI retrospective creates deliberate space for the proactive governance that general retrospectives tend to defer.

**Proposed Solution:**
- Hold a dedicated monthly AI practice review (30 minutes, architect-led): cover AI code percentage, rework rate, security vulnerability trend, and any session efficiency signals from engineer logs. This is separate from the general sprint retrospective.[^1]
- Hold a quarterly engineering health review (60 minutes, architect presents to CTO): cover codebase health indicators, CLAUDE.md currency, prompt library effectiveness, and year-on-year trend in all AI governance metrics. This is the session where governance decisions are made at the leadership level.[^14]
- After any incident or high-rework sprint, conduct a targeted retrospective focused specifically on whether AI governance practices contributed to the problem: was the CLAUDE.md context insufficient? Were verification standards skipped? Did a task category outside the delegable 20% get AI-primary treatment?[^5]
- Track retrospective action items to completion: a retrospective that generates recommendations but no closed action items is theater. Assign each action item an owner and a due date, and verify completion at the following session.[^12]

---

## Summary of Recommended Actions

| Metric Area | Immediate Action | Owner |
|---|---|---|
| AI Code Volume | Add AI-assisted flag to PR template; set 40% threshold alert | Architect |
| Rework Rate | Define rework in tracking system; add to monthly CTO report | Architect |
| Security Vulnerability Trends | Configure SAST output to central tracking; add trend to health review | Backend lead |
| Session and Context Efficiency | Start engineer session log; monthly aggregate review | Engineering team |
| Codebase Health | Track four health indicators monthly; schedule quarterly deprecation sprint | Architect |
| Retrospective Cadence | Schedule monthly practice review + quarterly health review | Architect + CTO |

---

[^1]: DEV Community — "AI Is Creating a New Kind of Tech Debt — And Nobody Is Talking About It," March 2026. https://dev.to/harsh2644/ai-is-creating-a-new-kind-of-tech-debt-and-nobody-is-talking-about-it-3pm6
 Velocity metrics as insufficient governance: the argument for a second measurement layer tracking rework, quality, and comprehension alongside output volume.

[^2]: GitHub — "Octoverse 2025: The State of Open Source and AI on GitHub," GitHub, 2025. https://github.blog/news-insights/octoverse/octoverse-2025/
 DORA metrics and AI adoption: 154% increase in PR size, 91% more review time, and 7.2% decrease in delivery stability correlated with high AI code generation rates.

[^3]: Kyros — "The Vibe Coding Crisis: How AI-Generated Technical Debt Is Costing Companies Millions," March 2026. https://usekyros.ai/blog/vibe-coding-crisis-ai-technical-debt
 Rework rate increase of 30–60% within six months of heavy AI adoption; the rework accumulation pattern and the compounding cost model that shows the inflection point for a 10-person team.

[^4]: Sonar (SonarSource) — "Sonar Data Reveals Critical 'Verification Gap' in AI Coding," press release, January 8, 2026. https://www.sonarsource.com/company/press-releases/sonar-data-reveals-critical-verification-gap-in-ai-coding/
 42% of all committed code originating from AI; the verification gap between stated distrust and actual verification behavior; baseline data for tracking AI code percentage.

[^5]: Anthropic — "2026 Agentic Coding Trends Report," Anthropic, 2026. https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf
 The 40% threshold: above 40% AI-primary code per sprint, rework rates spike materially; the sustainable ratio range and its implications for sprint planning and governance.

[^6]: GitClear — "2025 Coding on Copilot: 2023 Data Shows Downward Pressure on Code Quality," GitClear Research, 2025. https://www.gitclear.com/coding_on_copilot_data_shows_ais_downward_pressure_on_code_quality
 Copy-paste code rising from 8.3% to 12.3%; refactoring collapsing from 25% to under 10% of commits; longitudinal data on how AI adoption changes codebase health indicators.

[^8]: Veracode — "Spring 2026 GenAI Code Security Update: Despite Claims, AI Models Are Still Failing Security," March 24, 2026. https://www.veracode.com/blog/spring-2026-genai-code-security/
 Security pass rate stagnant at 55% despite model capability improvements; the argument that model upgrades do not solve the security vulnerability problem — practice changes are required.

[^9]: Dark Reading — "AI-Generated Code Poses Security, Bloat Challenges," October 2025. https://www.darkreading.com/application-security/ai-generated-code-leading-expanded-technical-security-debt
 Vulnerability category tracking: why aggregate counts obscure actionable signals; the case for tracking injection, secrets management, and API exposure trends separately.

[^10]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
 CLAUDE.md freshness tracking as a proxy for context health; using vulnerability trends to trigger CLAUDE.md updates; session hygiene metrics (/compact usage, correction cycle patterns).

[^12]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
 Session correction cycles as a feedback signal for prompt improvement; the discipline of converting session failures into CLAUDE.md and prompt library updates.

[^13]: Yue Liu et al. — "Debt Behind the AI Boom: A Large-Scale Empirical Study of AI-Generated Code in the Wild," arXiv:2603.28592, March 30, 2026. https://arxiv.org/html/2603.28592
 24.2% of AI-introduced code quality issues survive to the latest repo revision; the survival rate as a codebase health indicator and its interpretation for small teams.

[^14]: CIO — "How Agentic AI Will Reshape Engineering Workflows in 2026," April 2026. https://www.cio.com/article/4134741/how-agentic-ai-will-reshape-engineering-workflows-in-2026.html
 Retrospective cadence as governance infrastructure: the organizations achieving consistent improvement share structured regular review — AI quality metrics are only as valuable as the decisions they inform.

[^17]: Sabrina Ramonov — "CLAUDE CODE FULL COURSE," YouTube, February 17, 2025. https://www.youtube.com/watch?v=fYX6hHC9FhQ
 - Quality gate metrics: using hook-driven CI output as the raw data for security vulnerability trend tracking
 - Session logs: how to structure qualitative session outcome logs for monthly aggregate analysis
 - CLAUDE.md effectiveness testing: using session correction rates to evaluate whether context configuration is current and complete
