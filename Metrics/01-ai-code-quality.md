## AI Code Quality Metrics: Measuring What Matters After AI Generation

**Related to:** [Metrics Overview](00-overview.md) — Metric 1 · [Issues: Architectural Drift](../Issues/03-architectural-drift.md)[^a] · [Governance: Review Policies](../Governance/01-review-policies.md)[^b] · [Tooling: CI/CD Integration](../Tooling & Configuration/06-cicd-integration.md)[^c] · [QA & Testing: AI-Generated Test Coverage](../QA%20%26%20Testing/02-ai-generated-test-coverage.md)[^d]

---

## Overview

AI-generated code has a distinctive quality profile that differs from human-written code in measurable ways. CodeRabbit's analysis of 470 GitHub pull requests found that AI-generated PRs had logic and correctness issues 75% more common and readability problems 3× more prevalent than human-authored ones.[^1] Veracode's Spring 2026 security analysis found that 45% of AI-generated code fails security tests — a rate unchanged despite significant model capability improvements.[^2] These are industry-wide baselines; the team's specific rate depends on its prompting practices, CLAUDE.md configuration, and verification standards. Measuring the team's rate — and tracking it over time — is the mechanism by which governance produces improvement rather than just documentation.

This memo covers the specific metrics that reveal AI code quality, how to collect them without adding significant engineering overhead, the threshold values that should trigger intervention, and the diagnostic practices that convert metrics from surveillance into improvement. The goal is a measurement practice that makes quality problems visible early and makes improvements legible when they occur.

---

## Section 1: Defect Rate by Code Origin

**Description:** The most direct AI code quality metric is the defect rate differential: do PRs marked as AI-primary have a higher post-merge defect rate than human-authored PRs? If yes, by how much, and is the gap narrowing or widening over time? A team that has implemented strong verification practices and CLAUDE.md configuration should expect the defect rate differential to be lower than the industry average (1.7×); a team that has not should expect it to be higher.[^3]

Defect rate by code origin requires a two-part tracking system: PR origin classification (AI-primary, AI-assisted, human-authored) and defect attribution (which merged PR did this bug originate from?). The first is a PR template field; the second requires consistent use of `Fixes #issue` tags in commits so that bugs can be traced to the PRs that introduced them. Without both, the metric cannot be computed.[^4]

**Recommended Practice:**
- Add an AI origin field to the PR template: "Is this PR AI-primary (>50% AI-generated), AI-assisted (<50% AI-generated), or human-authored?" This self-reporting creates the classification data needed to compute defect rates by origin without automated analysis tools.[^3]
- Require the use of `Fixes #issue` or `Closes #issue` tags in bug-fix commits. Without these tags, tracking which PR introduced a defect requires manual investigation. With them, the relationship is automatically recorded and can be analyzed retroactively.[^4]
- Report the 90-day rolling defect rate differential at the monthly AI practice review. "AI-primary PRs are producing defects at 1.8× the rate of human-authored PRs" is a specific, actionable metric. "We have some quality concerns about AI code" is not.[^5]
- When the defect rate differential exceeds 2× for two consecutive months, treat it as a governance flag: convene a targeted retrospective to identify whether the issue is prompting practice, CLAUDE.md staleness, verification shortcuts, or a change in the types of tasks being AI-generated.[^2]

---

## Section 2: Security Finding Trends

**Description:** Security findings from automated scanning (SAST, dependency analysis, secrets detection) provide the most objective quality signal for AI-generated code because they are not subject to reviewer attention or review-time fatigue. A scanner that finds a vulnerability finds it regardless of whether the reviewer noticed it; a scanner that does not find a vulnerability provides a baseline assurance level regardless of how long the reviewer spent.[^6]

The trending dimension is what makes security metrics governable rather than just alarming. A single sprint with high critical findings may be noise; three consecutive sprints with rising critical findings is a signal. A correlation between rising AI code percentage and rising security findings confirms that AI adoption is outpacing governance; a stable or declining finding rate alongside rising AI adoption confirms that governance is keeping pace.[^7]

**Recommended Practice:**
- Run SAST scanning on every PR and export findings to a central tracking system (a spreadsheet or lightweight database is sufficient for a small team). Record: PR origin (AI/human), finding severity, finding category (injection, secrets, authentication, etc.), and whether it was caught pre-merge (blocking) or post-merge (audit).[^6]
- Calculate two monthly metrics: **pre-merge blocking rate** (proportion of AI-primary PRs blocked by SAST findings before merge) and **post-merge finding rate** (proportion of merged AI-primary PRs found to have security issues in subsequent review or incident). Rising post-merge rates indicate that pre-merge scanning is missing issues; rising pre-merge rates indicate that AI generation is producing more issues.[^7]
- Break findings down by category monthly: injection, secrets management, authentication, API exposure, deserialization. Category distribution is more actionable than aggregate counts — a team whose injection count is declining but secrets management findings are rising has a specific practice gap to address.[^2]
- Use rising security finding trends as a trigger for CLAUDE.md updates: a finding category that appears repeatedly in AI-generated code across multiple engineers is a candidate for a CLAUDE.md prohibition. "Never hardcode credentials or API keys" added to CLAUDE.md is more reliable than continuing to catch the same pattern in CI review.[^8]

---

## Section 3: Rework Rate Attribution

**Description:** Rework rate — the proportion of merged code requiring modification within 30 days due to defects, architectural misfit, or comprehension-driven rewrites — is the primary indicator of whether AI governance is keeping pace with AI output velocity. Raw rework rate measures total post-merge instability; rework rate attribution separates the portion attributable to AI-primary PRs from the portion attributable to human-authored PRs, enabling the comparison that reveals whether AI code is producing disproportionate rework.[^9]

The METR productivity studies found that developers sometimes experienced productivity slowdowns on complex tasks despite expected speedups — and that the slowdowns were disproportionately concentrated in rework following AI-generated implementations that encountered implicit decision points. Rework attribution data reveals this pattern at the team level: if AI-primary PRs are generating 30% of merged code but 60% of rework, the net productivity gain from AI adoption is significantly lower than velocity metrics suggest.[^10]

**Recommended Practice:**
- Define rework in the tracking system with specific criteria: a commit is rework if it (a) modifies code merged in the prior 30 days AND (b) the modification was motivated by a defect, an architectural correction, or a comprehension-driven rewrite (not planned iteration). Without a clear definition, classification is inconsistent and the metric is unreliable.[^9]
- Track the rework origination PR using commit tags: "rework: #123" in a rework commit's message establishes the relationship between the rework commit and the PR that necessitated it. This enables the rework rate by origin analysis that makes the metric actionable.[^4]
- Present the rework rate differential (AI-primary vs. human-authored) alongside the velocity differential at the quarterly engineering health review. A team reporting "AI code is 2× faster to generate but 1.8× more likely to require rework" has the complete picture for the CTO to make an informed governance decision.[^5]
- If rework rate is rising but security findings are not, the rework is likely architectural or comprehension-driven rather than security-related. Investigate whether CLAUDE.md context is sufficient for the affected modules and whether the AI code percentage in those modules has recently increased.[^11]

---

## Section 4: Test Coverage Trends for AI-Generated Code

**Description:** AI-generated code tends to be accompanied by tests generated in the same session — and tests generated in the same session as the code they test have a structural weakness: they were designed with knowledge of the implementation, which biases them toward testing what the code actually does rather than what it should do. A test suite for an AI-generated feature may have 90% line coverage while missing the behavioral edge cases that the implementation silently handles incorrectly.[^12]

Coverage metrics for AI-generated code therefore require interpretation beyond raw percentages. The relevant questions are: were the tests written in the same session as the implementation (same-session tests have lower behavioral coverage on average)? Do the tests cover the function's specified behavior or only its implemented behavior? Were edge cases explicitly requested in the test prompt or left to AI inference?[^13]

**Recommended Practice:**
- Track test coverage separately for AI-primary vs. human-authored modules in the monthly codebase health indicators. If AI-primary modules have systematically lower coverage, the team's test generation prompts need updating to be more explicit about coverage requirements.[^12]
- Flag PRs where the test coverage for new code is below 80% line coverage AND the code was AI-primary. This combination warrants specific reviewer attention: either the AI-generated tests are inadequate or the code itself is not sufficiently bounded for effective testing.[^6]
- For critical modules (authentication, payment processing, data access), require test generation in a separate session from code generation — using the writer/reviewer pattern for tests (see Prompting — Task Patterns). Same-session test generation for critical code should be treated as a yellow flag, not an acceptable default.[^13]
- After each incident that was caused by a bug in AI-generated code that had test coverage, analyze whether the coverage was adequate or whether the tests were testing implementation rather than behavior. This retrospective feeds into both test prompt improvement and CLAUDE.md updates for testing requirements.[^8]

---

## Section 5: Code Complexity and Duplication Trends

**Description:** AI tools generate code that works before they generate code that is elegant, minimal, or architecturally consistent with its surroundings. GitClear's longitudinal analysis found that AI adoption correlated with copy-paste code rising from 8.3% to 12.3% of all code and refactoring collapsing from 25% to under 10% of commits.[^14] These trends compound: code that is not refactored accumulates complexity and duplication, which makes future AI sessions in that area more likely to produce inconsistent outputs (because the area they are working in is inconsistent), creating a feedback loop where poor codebase health degrades AI output quality.

Complexity and duplication metrics require tooling to measure automatically — tools like SonarQube, CodeClimate, or language-specific linters provide these metrics as CI outputs — but require human interpretation to act on. Rising complexity in specific modules identifies areas where AI-generated code is accumulating technical debt; rising duplication identifies areas where AI is re-implementing rather than reusing.[^15]

**Recommended Practice:**
- Track four codebase health indicators monthly: total lines of code growth rate, duplicate code percentage, average cyclomatic complexity per module, and test coverage trend. Present these as a trend rather than a point-in-time snapshot — the direction matters more than the absolute value.[^14]
- Set alert thresholds: if any single module's cyclomatic complexity increases by more than 20% in a quarter, investigate whether AI-generated code in that module is the root cause. If duplicate code percentage rises above 15%, schedule a deprecation sprint to address the accumulation.[^15]
- Include the architect in reviewing complexity and duplication trends monthly: these are architectural health indicators that require architectural judgment to interpret and address. A backend engineer can identify that complexity is rising; the architect can determine whether the root cause is a prompting problem, a CLAUDE.md gap, or a task type issue.[^11]
- Use complexity metrics as input for the AI readiness classification in sprint planning: modules with high and rising complexity are poor candidates for AI-primary implementation until the complexity is reduced. Sending AI into a highly complex module without explicit architectural context tends to produce output that makes the complexity worse.[^9]

---

## Summary of Recommended Practices

| Metric | Immediate Action | Owner |
|---|---|---|
| Defect Rate by Origin | Add AI origin field to PR template; implement defect attribution tagging | Architect |
| Security Finding Trends | Configure SAST output to central tracking; add category breakdown to monthly review | Backend lead |
| Rework Rate Attribution | Define rework criteria; add rework tagging to commit convention | Architect |
| Test Coverage Trends | Track coverage by module origin; flag AI-primary PRs below 80% | QA + Architect |
| Complexity and Duplication | Configure automated complexity tracking in CI; set alert thresholds | Architect |

---

[^1]: CodeRabbit — "State of AI Code Generation: AI vs. Human Code Report," December 17, 2025. https://www.coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report
    AI-generated PRs with 75% more logic/correctness issues and 3× readability problems; root cause analysis (statistical inference vs. semantic understanding) as baseline for quality metric design.

[^2]: Veracode — "Spring 2026 GenAI Code Security Update: Despite Claims, AI Models Are Still Failing Security," March 24, 2026. https://www.veracode.com/blog/spring-2026-genai-code-security/
    45% AI code security failure rate; stagnation as evidence that model improvements do not solve the quality problem — practice and measurement changes are required.

[^3]: daily.dev — "Vibe Coding in 2026: How AI Is Changing the Way Developers Write Code," April 2026. https://daily.dev/blog/vibe-coding-how-ai-changing-developers-code
    1.7× defect rate differential for AI-generated code as the industry baseline; team-specific rate as a function of governance practice quality.

[^4]: GitHub — "Octoverse 2025: The State of Open Source and AI on GitHub," GitHub, 2025. https://github.blog/news-insights/octoverse/octoverse-2025/
    Commit tagging patterns for defect attribution; PR metadata as the foundation for origin-based quality analysis.

[^5]: Anthropic — "2026 Agentic Coding Trends Report," Anthropic, 2026. https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf
    Metric presentation cadence: monthly practice review vs. quarterly health review as the two governance feedback loops that make quality metrics actionable at different time horizons.

[^6]: Dark Reading — "AI-Generated Code Poses Security, Bloat Challenges," October 2025. https://www.darkreading.com/application-security/ai-generated-code-leading-expanded-technical-security-debt
    SAST scanning as the objective quality signal: why automated scanning is more consistent than reviewer attention; pre-merge vs. post-merge finding rates as the two security quality metrics.

[^7]: Kyros — "The Vibe Coding Crisis: How AI-Generated Technical Debt Is Costing Companies Millions," March 2026. https://usekyros.ai/blog/vibe-coding-crisis-ai-technical-debt
    Security finding trends: the difference between single-sprint noise and multi-sprint signal; the correlation between rising AI percentage and rising security findings as a governance flag.

[^8]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    CLAUDE.md updates driven by quality metrics: how recurring security finding categories become CLAUDE.md prohibitions; the feedback loop from metrics to configuration improvement.

[^9]: DEV Community — "AI Is Creating a New Kind of Tech Debt — And Nobody Is Talking About It," March 2026. https://dev.to/harsh2644/ai-is-creating-a-new-kind-of-tech-debt-and-nobody-is-talking-about-it-3pm6
    Rework rate as the net productivity metric: how rework rate attribution reveals the true cost of AI adoption beyond raw velocity numbers.

[^10]: METR — "We Are Changing Our Developer Productivity Experiment Design," METR Research, February 2026. https://www.metr.org/blog/2026-02-24-uplift-update/
    Productivity slowdown from rework: the 19% slowdown data point and its relationship to rework accumulation in AI-generated implementations that encountered implicit decision points.

[^11]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    CLAUDE.md freshness as a quality metric: how correlation between CLAUDE.md staleness and rising defect rates reveals the configuration-quality relationship that drives practice changes.

[^12]: Fannar Steinn Aðalsteinsson et al. — "Rethinking Code Review Workflows with LLM Assistance: An Empirical Study," arXiv:2505.16339, May 22, 2025. https://arxiv.org/abs/2505.16339
    Same-session test generation bias: how tests generated with knowledge of the implementation tend to test implemented behavior rather than specified behavior, producing coverage metrics that overstate behavioral assurance.

[^13]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
    Writer/reviewer pattern for test generation: separate-session tests for AI-generated code as the mechanism for achieving behavioral rather than implementation-biased coverage.

[^14]: GitClear — "2025 Coding on Copilot: 2023 Data Shows Downward Pressure on Code Quality," GitClear Research, 2025. https://www.gitclear.com/coding_on_copilot_data_shows_ais_downward_pressure_on_code_quality
    Copy-paste rate and refactoring collapse: longitudinal data on how AI adoption changes codebase health indicators; the feedback loop where poor codebase health degrades future AI output quality.

[^15]: Sonar (SonarSource) — "Sonar Data Reveals Critical 'Verification Gap' in AI Coding," press release, January 8, 2026. https://www.sonarsource.com/company/press-releases/sonar-data-reveals-critical-verification-gap-in-ai-coding/
    Automated quality tool configuration: how SonarQube and equivalent tools provide the complexity, duplication, and coverage metrics needed for codebase health tracking without manual measurement.


[^18]: Sabrina Ramonov — "CLAUDE CODE FULL COURSE," YouTube, February 17, 2025. https://www.youtube.com/watch?v=fYX6hHC9FhQ
    - Quality gate metrics: how to configure CI output for central tracking of security findings, complexity trends, and coverage rates by PR origin
    - CLAUDE.md feedback loop: using quality metrics to identify recurring failure modes and convert them into CLAUDE.md prohibitions and requirements
    - Rework visibility: how session logs and commit attribution create the data infrastructure for rework rate analysis without significant additional engineering overhead

[^a]: [Issues: Architectural Drift](../Issues/03-architectural-drift.md) — code quality metrics include architectural consistency signals; drift is measured and surfaced through quality metrics before it becomes structural.

[^b]: [Governance: Review Policies](../Governance/01-review-policies.md) — review policy pass rates and post-merge defect rates are the primary code quality metrics; the metrics operationalize how well review policy is functioning.

[^c]: [Tooling: CI/CD Integration](../Tooling & Configuration/06-cicd-integration.md) — CI/CD pipelines generate the quality signal data that the health dashboard aggregates; CI/CD integration is the data collection infrastructure for quality metrics.

[^d]: [QA & Testing: AI-Generated Test Coverage](../QA%20%26%20Testing/02-ai-generated-test-coverage.md) — test coverage is a primary code quality metric for AI-generated code; coverage analysis feeds the quality signal that this document defines.
