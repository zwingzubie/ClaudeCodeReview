## Codebase Bloat: The Expanding Surface Area Nobody Signed Up to Maintain

**Related to:** [Issues Overview](overview.md) — Issue 2 · [Metrics: Codebase Health](../Metrics/06-codebase-health.md)[^a] · [QA & Testing: Regression Prevention](../QA%20%26%20Testing/04-regression-prevention.md)[^b] · [Governance: Review Policies](../Governance/01-review-policies.md)[^c] · [Tooling: CLAUDE.md Configuration](../Tooling & Configuration/01-claude-md-configuration.md)[^d]

---

### What Is Codebase Bloat?

Codebase bloat in the AI era is distinct from classic technical debt. Traditional debt is code that works but was built quickly and needs cleaning up. AI-era bloat is different: it is code that works, was built quickly, and will never be cleaned up — because the very tools generating it at scale are systematically worse at removing, restructuring, and abstracting it than human developers are.

The mechanism is straightforward. AI tools are optimized to generate functional outputs, not minimal ones. They do not internalize team-specific architecture, existing abstractions, or the concept of "this already exists somewhere else in the codebase." Each new feature request produces a fresh implementation rather than an extension of what is already there. Over time, the codebase grows in volume while its coherence erodes. More code means more to read, more to maintain, more surface area for bugs, and longer onboarding for every new engineer who joins.

---

### The Data on AI-Driven Code Growth

GitClear's January 2026 analysis of 211 million changed lines of code across repositories owned by Google, Microsoft, Meta, and enterprise firms documents the shape of this problem with unusual precision.[^1]

Between 2021 and 2024:

- **Copy-pasted code rose from 8.3% to 12.3%** of all changed lines — approximately a **4x growth** in cloned code blocks
- **Refactoring (moved and restructured lines) collapsed from 25% to under 10%** of all changed lines
- For the first time in GitClear's measurement history, **copy-paste volume exceeded moved code** — meaning developers are adding duplicates faster than they are reusing or reorganizing existing logic
- **Code churn** (new code revised within two weeks of commit) rose from 3.1% to 5.7% — evidence that AI-generated code requires more immediate rework than human-written code

Faros AI's telemetry across 10,000 developers on 1,255 engineering teams corroborates this from the review side: AI adoption correlates with a **154% increase in pull request size** and a **9% increase in bug rates**.[^3] Engineers are merging larger, buggier PRs and spending 91% more time on code review to process them.

These numbers describe a compounding dynamic. Larger PRs mean less rigorous review. Less rigorous review means more defects. More defects mean more churn. And more churn means the codebase is growing even faster, as new "fixes" are layered over the original AI-generated code rather than replacing it.

---

### Why AI Tools Resist Refactoring

The collapse of refactoring is not incidental. It is structural.

An ACM peer-reviewed study (2025) evaluated large language model refactoring capability across 30 open-source Java projects. The finding was nuanced: AI tools could handle **systematic, surface-level refactoring** — removing magic numbers, shortening long statements — but **performed poorly on complex, context-dependent structural refactoring** — the kind that actually prevents bloat from accumulating.[^7] The operations that matter most for codebase health (Extract Class, Rename Module, Consolidate Hierarchy) are precisely the operations AI agents are worst at.

A separate systematic literature review of 50 primary studies on LLM-assisted refactoring (published in the *Journal of Systems and Software*, 2025) reached a consistent conclusion: "LLMs often generate erroneous code, struggle with more complex refactoring, and frequently misunderstand developer intent." No standardized measure of refactoring accuracy exists across studies, meaning industry claims about AI refactoring capability are systematically overstated.[^8]

The agentic refactoring study (arXiv, 2025) analyzed 15,451 refactoring instances performed by AI coding agents. The top operations were cosmetic: Change Variable Type (11.8%), Rename Parameter (10.4%), Rename Variable (8.5%). Median class size reduction: 15 lines. These interventions are **insufficient to counteract the 4x duplication growth** documented by GitClear in the same period.[^13]

Ox Security's analysis of 300 open-source repositories found "Avoidance of Refactors" appearing in 80–90% of AI-generated codebases. AI tools actively resist structural improvements — not because they are incapable in principle, but because generating fresh, functional code is always the path of least resistance.[^4]

---

### The Velocity Illusion and the True Cost

The METR randomized controlled trial — 16 experienced open-source developers, 246 real tasks — found that developers using AI tools (including Claude 3.5/3.7 and Cursor Pro) completed tasks **19% slower** than those working without AI. The slowdown was most pronounced in large, complex, mature codebases: exactly the kind of codebases that AI tooling has been expanding.[^2]

This is the velocity illusion in action. Early adoption of AI tools produces real speedups in greenfield work. But as the codebase grows — padded with duplicated code, under-refactored modules, and AI-generated boilerplate that no engineer fully owns — the productivity gains erode. The MIT Sloan Management Review's reporting on AI coding economics found that unmanaged AI-generated code can drive **maintenance costs to 4x traditional levels by year two**.

A Harness survey of 900 engineers and technical managers found that 72% of organizations have experienced at least one production incident directly caused by AI-generated code, and 45% of deployments involving AI-generated code introduce problems. The resulting gap — faster code generation without equivalent downstream automation — is what the report terms the "AI Velocity Paradox."[^6]

---

### The Dead Code Problem

Beyond duplication, AI-assisted development has a dead code accumulation problem. An ISSRE 2025 study comparing 500,000+ code samples from human developers vs. three LLMs found that AI-generated code shows greater prevalence of **unused constructs and hardcoded debugging statements** — a direct dead-code accumulation mechanism.[^10]

Meta's analysis of their own code improvement practices documents what systematic dead code removal actually delivers: a **90% decrease in severity-causing diffs** (odds ratio 5.2), with dead code removal showing the strongest reliability improvement of any code quality activity.[^11] Meta devotes over 14% of all code changes to explicit improvement activities — compared to a 4% industry average. The gap between what Meta invests in code health and what most teams invest is the gap between manageable debt and compounding decay.

DX's April 2026 research on developer ramp-up time found that the onboarding metric (time to 10th PR) has fallen 64% with AI adoption — from 91 days to 33 days. The researchers are explicit about the limitation: "This metric does not tell us about the quality of those changes, the amount of rework they generate, or the depth of understanding new hires have."[^12] Faster onboarding into a bloated codebase is not a win.

---

### Proposed Solutions

**1. Track Codebase Size as an Engineering Metric**

Establish explicit targets for codebase size relative to team size, and track the ratio over time. A codebase growing 20% per quarter while team size stays flat is a signal that bloat is compounding. Track duplication density, refactor ratio, and churn rate alongside velocity metrics. GitClear's methodology provides a usable framework.[^1]

**2. Monthly Codebase Health Review**

One hour, architect-led, focused specifically on identifying duplication, orphaned modules, and patterns that have diverged from established conventions. The goal is not to fix everything but to maintain visibility. Bloat accumulates invisibly — the health review makes it visible before it becomes unmanageable.

**3. Require an AI Refactor Pass Before Merging**

Before finalizing any AI-generated implementation, engineers should explicitly prompt the AI to refactor: "Before I commit this, review it for duplication, redundant logic, and any abstractions that already exist in the codebase." This is not automatic. It must be a named step in the PR process. The ACM and JSS research both confirm that AI does not volunteer refactoring — it must be asked.[^7][^8]

**4. Deprecation Sprints on the Product Roadmap**

Assign the architect explicit authority and calendar time to conduct focused deprecation efforts — removing or consolidating code, not adding features. These should appear on the roadmap as first-class items. The Meta research quantifies the payoff: dead code removal delivers more reliability improvement per developer-hour than almost any other code quality activity.[^11]

**5. Set a Pre-PR Refactoring Prompt as a Team Standard**

Add this to the PR template or team coding standards: *"Did you ask the AI to identify duplication before finalizing?"* Make refactoring for efficiency a named team expectation, not an individual judgment call.

---

### Summary

AI tools generate code at a rate that outpaces any team's capacity to review, refactor, and remove it. The result is systematic bloat: duplication grows, refactoring falls, dead code accumulates, and the codebase becomes progressively more expensive to understand, modify, and onboard into. On a small team, this cost is not distributed — it falls on the architect and the senior engineers most likely to already be operating at capacity. Treating codebase size as an unmanaged side effect of AI adoption is a governance failure. Treating it as a first-class engineering metric is the correction.

---

[^1]: GitClear (William Harding) — "AI Copilot Code Quality: 2025 Data Suggests 4x Growth in Code Clones," January 2026. https://www.gitclear.com/ai_assistant_code_quality_2025_research
 Analysis of 211 million changed lines from Google, Microsoft, Meta, and enterprise repos. Copy-pasted code rose from 8.3% to 12.3% of changed lines (2021–2024). Refactoring fell from 25% to under 10%. First time in GitClear history that copy-paste volume exceeded moved/reused code.

[^2]: Joel Becker, Nate Rush, Elizabeth Barnes, David Rein (METR) — "Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity," arXiv:2507.09089, July 2025. https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/
 Randomized controlled trial with 16 experienced developers across 246 real tasks. AI tools (Cursor Pro + Claude 3.5/3.7) produced a 19% slowdown vs. no-AI control group. Slowdown most pronounced in large, complex, mature codebases.

[^3]: Faros AI — "DORA Report 2025 Key Takeaways: AI Impact on Dev Metrics," 2025. https://www.faros.ai/blog/key-takeaways-from-the-dora-report-2025
 Telemetry across 10,000 developers on 1,255 teams. AI adoption correlates with +98% more PRs merged, +154% increase in PR size, +91% increase in code review time, and +9% increase in bug rates. Individual productivity gains do not translate to organizational delivery improvements. The 2025 DORA report (dora.dev/research/2025/dora-report/) separately confirmed AI acts as an amplifier of existing team strengths and weaknesses across ~5,000 respondents.

[^4]: Ox Security — "The Army of Juniors: The AI Code Security Crisis," October 2025. https://www.ox.security/resource-category/whitepapers-and-reports/army-of-juniors/
 Analysis of 300+ open-source repositories. "Avoidance of Refactors" appears in 80–90% of AI-generated codebases. AI generates highly functional but architecturally judgment-free code.

[^6]: Harness Inc. — "The State of AI in Software Engineering," January 2026. https://www.harness.io/the-state-of-ai-in-software-engineering
 Survey of 900 engineers, platform leaders, and technical managers (conducted August 2025). 72% of organizations experienced at least one production incident from AI-generated code. 45% of AI code deployments introduce problems. 48% worry about increased vulnerabilities. Identifies the "AI Velocity Paradox": code generation accelerates but downstream testing, security, and deployment automation has not kept pace.

[^7]: Jonathan Cordeiro, Shayan Noei, Ying Zou (Queen's University) — "An Empirical Study on the Code Refactoring Capability of Large Language Models," ACM Transactions on Software Engineering and Methodology, 2025. https://arxiv.org/abs/2411.02320
 Evaluated LLM refactoring across 30 Java projects. LLMs excel at systematic surface-level refactoring but perform poorly on complex context-dependent structural changes — the operations most relevant to preventing bloat.

[^8]: Sofia Martinez et al. — "Software Refactoring Research with Large Language Models: A Systematic Literature Review," *Journal of Systems and Software*, Vol. 235, May 2026. https://doi.org/10.1016/j.jss.2025.112762
 Systematic review of 50 primary studies. LLMs frequently generate erroneous code, struggle with complex refactoring, and misunderstand developer intent. No standardized accuracy definition exists — reliability of AI refactoring claims is overstated industry-wide.

[^9]: H. Al-Fawareh, H.M. Al-Shdaifat — "Investigating AI-Generated Code on the Impact on Software Efficiency Code Quality Factor," Springer, 2025. https://doi.org/10.1007/978-3-031-83911-5_8
 Compared ChatGPT-generated code to GitHub human code. AI-generated code shows 34% greater cyclomatic complexity and 2.1x greater code duplication vs. human-written code.

[^10]: Domenico Cotroneo, Cristina Improta, Pietro Liguori — "Human-Written vs. AI-Generated Code: A Large-Scale Study," arXiv:2508.21634, IEEE ISSRE 2025. https://arxiv.org/abs/2508.21634
 Compared 500,000+ code samples across Python and Java. AI-generated code is "generally simpler and more repetitive" with greater prevalence of unused constructs and hardcoded debugging statements — a direct dead-code accumulation mechanism.

[^11]: Audris Mockus et al. (Meta) — "Code Improvement Practices at Meta," arXiv:2504.12517, April 2025. https://arxiv.org/abs/2504.12517
 Dead code removal showed a 90% decrease in severity-causing diffs (odds ratio 5.2) — the strongest reliability improvement of any code quality activity. Meta devotes 14%+ of all changes to improvement; industry average is 4%.

[^12]: Justin Reock (DX) — "Developer Ramp-Up Time Continues to Accelerate with AI," April 9, 2026. https://newsletter.getdx.com/p/developer-ramp-up-time-continues
 400 companies, 500+ developers each. Time to 10th PR fell from 91 days to 33 days (64% reduction). Critical caveat: metric "does not tell us about the quality of those changes, the amount of rework they generate, or the depth of understanding new hires have."

[^13]: Kosei Horikawa et al. — "Agentic Refactoring: An Empirical Study of AI Coding Agents," arXiv:2511.04824, November 2025. https://arxiv.org/abs/2511.04824
 Analyzed 15,451 AI refactoring instances across Java projects. Top operations are cosmetic (variable renaming, type changes). Median class LOC reduction: 15 lines — insufficient to counteract the 4x duplication growth documented concurrently. Agents preferentially avoid the complex structural refactoring most important for codebase health.

[^a]: [Metrics: Codebase Health](../Metrics/06-codebase-health.md) — codebase health indicators are the primary measurement mechanism for bloat; the metrics defined there operationalize the risks described here.

[^b]: [QA & Testing: Regression Prevention](../QA%20%26%20Testing/04-regression-prevention.md) — regression prevention is more expensive as the codebase surface area grows; bloat and regression risk compound each other.

[^c]: [Governance: Review Policies](../Governance/01-review-policies.md) — review policies include scope gates that limit AI-generated additions per PR; these are the primary governance mechanism for controlling bloat rate.

[^d]: [Tooling: CLAUDE.md Configuration](../Tooling & Configuration/01-claude-md-configuration.md) — CLAUDE.md constraints can prohibit specific patterns that generate bloat; the configuration layer is the session-time enforcement point for scope discipline.
