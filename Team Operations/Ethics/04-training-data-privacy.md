## Training Data and Privacy: Managing Sensitive Information in AI Sessions

**Related to:** [Ethics Overview](00-overview.md) — Risk 4 · [Governance: AI Usage Policy](../Governance/02-ai-usage-policy.md)[^a] · [Security: Secrets Management](../Security/04-secrets-management.md)[^b] · [Tooling: Settings and Permissions](../Tooling & Configuration/05-settings-and-permissions.md)[^c] · [Governance: Compliance and Audit](../Governance/06-compliance-and-audit.md)[^d]

---

## Overview

Every Claude Code session generates data: the prompts engineers submit, the context they provide, the code they paste, the error messages they share. That data is transmitted to Anthropic's infrastructure and processed to generate responses. The question engineers rarely ask — but should ask before each session — is: what data am I including in this session, and under what terms will it be retained and used? The gap between what engineers believe about their session data and what the applicable terms actually say is one of the least-examined privacy risks in AI-assisted development.[^1]

This is not a hypothetical concern for a small team working on real products with real customers. Customer data in debugging sessions, unreleased feature details in design assistance sessions, PII in error log analysis — these are the normal artifacts of a development workflow applied to a live product. Each of them represents data that was produced under a privacy obligation or a competitive confidentiality expectation, being included in an AI session whose data handling terms may not meet those obligations. The practical risk is not that Anthropic is malicious — it is that engineers do not think about session data privacy at the moment of session composition, when the decisions that determine the risk are actually made.[^2]

---

## Section 1: How AI Training Data Creates Privacy Risk

**Description:** The data cycle for a Claude Code session runs from prompt composition through session processing through data retention and, under some plan configurations, potential training data contribution. The engineer composes a prompt — which may include code, error messages, user data, configuration values, or internal documentation — and submits it. The session data is processed and retained according to Anthropic's current data use policies and the team's plan type. Under some configurations, session data may contribute to future model training, which means that information submitted in a session today could, in theory, influence model outputs for users of future models.[^3]

The critical distinction is between what is logged and what is used for training. Anthropic logs session data for safety monitoring, quality improvement, and abuse prevention under all plan types. Training data contribution is a separate question governed by plan type and any applicable data use agreements. Engineers who assume their sessions are completely ephemeral are operating under an incorrect model of what happens to their session data — and that incorrect model leads to decisions about session content that they might make differently with accurate information.[^4]

**Recommended Practice:**
- Brief the full team on the distinction between session logging (which occurs under all plans) and training data contribution (which varies by plan and configuration). Engineers who understand the full data lifecycle make more careful session composition decisions.[^1]
- Add a session data awareness question to the team's development workflow checklist: "Does this session include data that was produced under a privacy obligation or competitive confidentiality expectation?" This question, asked before session composition, is the primary intervention point for preventing inadvertent privacy exposure.
- Maintain a written summary of the team's current Anthropic plan type and its data handling provisions, updated whenever the plan changes. This document should be accessible to all engineers, not just to the CTO who manages the account.[^3]
- Review the team's data handling provisions after any significant change to the product's data processing obligations — a new customer contract, a new regulatory requirement, a significant product feature change. What was adequate before the change may not be adequate after it.

---

## Section 2: Anthropic's Data Use Policies

**Description:** Anthropic's data use policies differentiate between individual API use, team API use, and enterprise API use, with different data retention, training data contribution, and data processing terms at each tier. For the Claude Code integration, the relevant policy document is Anthropic's Usage Policy and the applicable data processing addendum for the team's plan type. The policies were most recently substantively updated in late 2025 and apply to sessions generated through both the API and the Claude Code product.[^7]

Under the current policies, enterprise plan customers can negotiate data processing agreements that include training data opt-out, reduced retention periods, and data residency provisions. These provisions are not available by default under individual or standard team plans. The practical implication for a team working on a commercial product with customer data is that the default plan may not provide the data handling protections the team's obligations require — and that determining whether the current plan is adequate requires reading the current policy documentation, not relying on assumptions formed when the plan was first established.

**Recommended Practice:**
- The CTO should review the current Anthropic data use policy and data processing addendum annually, not just at initial account setup. Policy terms change, and annual review is the minimum frequency for maintaining an accurate understanding of the team's data handling posture.[^9]
- Determine whether the team's current plan type includes training data opt-out. If not, evaluate whether the team's data obligations require that provision and upgrade to an appropriate plan if so. This is a binary determination — the team either has the provision or does not.[^7]
- Document the outcome of the annual policy review in writing: what the current terms are, whether they meet the team's current obligations, and what changes (if any) were made to the plan or usage practices as a result. This documentation is the evidence of due diligence if a compliance question arises.
- Assign the annual review as a specific CTO calendar task rather than a general "review when relevant" obligation. Compliance reviews that are not on a calendar are not reliably performed — and a privacy incident is not the right trigger for discovering that the policy review has not occurred.[^3]

---

## Section 3: Customer Data in AI Sessions

**Description:** The most frequent source of inadvertent privacy exposure in AI-assisted development is the debugging and troubleshooting workflow: an engineer encounters a production issue, pulls an error log or a database record to understand it, and pastes it into a Claude session to get diagnostic assistance. The session now contains production customer data. The engineer's intent was diagnostic; the data handling implication was not part of the decision.[^10]

This pattern — real data in debugging sessions — is not unusual behavior by careless engineers. It is the natural result of applying AI assistance to real-world development tasks. The practical question is not how to prevent engineers from debugging production issues, but how to apply the data minimization principle to the way those sessions are composed: use the minimum customer data necessary to diagnose the issue, prefer synthetic or anonymized examples where possible, and recognize that real customer data in an AI session is a data handling event that should be deliberate rather than incidental.[^11]

**Recommended Practice:**
- Establish a data minimization standard for AI debugging sessions: use anonymized or synthetic data wherever the diagnostic question can be answered without real customer data. When real data is genuinely necessary, use the minimum subset required — not full records when partial records will do, not production data when staging data will do.
- Add a note to CLAUDE.md for production-adjacent modules: "When debugging sessions involve production data, apply data minimization. Do not include full customer records, PII fields, or payment data in session context." This reminder is visible at the point of session composition, when the decision is actually made.[^12]
- Define the categories of customer data that may not appear in AI sessions under any circumstances — payment card data, social security numbers, health records, authentication credentials — and brief the team on those prohibitions explicitly. Some data categories warrant an absolute rule rather than a judgment call.[^2]
- When a debugging session requires real customer data to diagnose an issue, document it: what data was used, what session it appeared in, what the diagnostic purpose was. This is the minimum record required to demonstrate that production data in AI sessions is tracked and deliberate rather than routine and unmonitored.[^1]

---

## Section 4: Pre-Release and Proprietary Information

**Description:** Competitive exposure through AI sessions is a less visible privacy risk than customer data — the information is confidential rather than regulated, and the harm is competitive rather than privacy-related — but it is a real risk for a team actively developing unreleased features. An engineer using Claude to design an unreleased API, discuss acquisition considerations, or prototype a roadmap feature is including competitively sensitive information in a session whose data handling may expose it beyond the team's control.[^13]

The risk is heightened for agentic sessions — multi-step sessions where Claude Code operates across multiple files and modules. Agentic sessions may encounter unreleased code, internal documentation, or configuration files that were not explicitly included in the engineer's prompt but were included by Claude's tool calls during the session. This means the information scope of an agentic session is broader than what the engineer consciously chose to include, and the privacy implications of that scope may not be fully visible at session start.

**Recommended Practice:**
- Maintain a list of information categories that require session scope awareness before use in AI sessions: unreleased features under active development, acquisition or partnership discussions, roadmap details not in public documentation, investor or board materials. This list is not a prohibition — it is a prompt to verify that the session's data handling terms are appropriate before including such information.[^13]
- For sessions involving unreleased product features, add a prompt-level reminder: "This session involves unreleased features. Limit context to what is necessary for the current task." This narrows the information scope deliberately rather than letting Claude's context-building behavior determine what proprietary information appears in the session.[^12]
- For agentic sessions in modules containing unreleased or proprietary features, configure the session scope to exclude directories or files whose contents should not appear in session data. Claude Code's CLAUDE.md `ignore` configuration is the mechanism for this scope control.
- Brief the team on the agentic session scope issue specifically: unlike a manual prompt where the engineer chooses what to include, an agentic session's information scope is determined partly by Claude's tool call decisions. Engineers who understand this difference apply more deliberate session scoping than those who assume agentic sessions contain only what they explicitly asked about.

---

## Section 5: Team Data Governance Practices

**Description:** The team's data governance practices for AI sessions need to be explicit, documented, and actively maintained — not because the risk is catastrophic on any given day, but because the risk accumulates across the team's daily AI usage and the gap between "what we assumed" and "what the policy says" tends to grow rather than shrink without active maintenance. A governance framework that was designed at AI adoption time and never revisited is a framework that does not reflect the team's current usage patterns, data obligations, or platform policy terms.[^15]

CLAUDE.md session scope reminders are the primary operational tool for maintaining data governance in the normal workflow. They work because they appear at the point of session composition, where the data handling decision is actually made — not in a policy document that was read at onboarding and may not have been read since. But CLAUDE.md reminders are only effective if they are accurate, current, and specific enough to be actionable. A vague reminder to "be careful with sensitive data" produces less behavior change than a specific reminder identifying the categories of data that require scope verification.[^16]

**Recommended Practice:**
- Implement a data classification framework for AI session content with three tiers: (1) unrestricted — general code, public documentation, synthetic test data; (2) sensitive — production-adjacent debugging data, internal architecture documentation, unreleased feature details; (3) prohibited — customer PII, payment data, credentials, regulated health information. Brief the team on the framework and add tier markers to the relevant modules' CLAUDE.md files.[^11]
- Add explicit CLAUDE.md data handling reminders to sensitive-tier modules: include the data classification, the applicable session data rules, and the data minimization instruction. These reminders should be maintained as the module's sensitivity profile changes.[^12]
- Run an annual data governance review for AI sessions alongside the Anthropic policy review: audit a sample of recent sessions (where accessible) against the data governance framework; identify gaps between intended and actual session data practices; update CLAUDE.md configurations and team training based on findings.[^9]
- Include AI session data governance in engineer onboarding: not just "what Claude Code is" but "what data handling obligations apply to your sessions, what data categories require special handling, and where to find the current CLAUDE.md configuration for sensitive modules." Engineers who receive this information at onboarding are better positioned to make deliberate data handling decisions than those who discover the issues after the fact.[^2]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Data Lifecycle Awareness | Brief team on logging vs. training data distinction; document current plan terms | CTO |
| Policy Review | Schedule annual Anthropic policy review; document outcome | CTO |
| Customer Data Minimization | Add data minimization reminders to production-adjacent CLAUDE.md files | Architect |
| Proprietary Information | Add session scope reminders to unreleased-feature modules; configure agentic scope | Architect |
| Data Governance Framework | Implement three-tier classification; add to engineer onboarding | CTO + Architect |

---

[^1]: Anthropic — "Data Usage and Privacy," Claude Code Documentation, 2026. https://code.claude.com/docs/en/data-usage
 Session data retention and training use policies: what is logged, what may contribute to training, and how plan type affects these terms; the distinction between logging and training data contribution.

[^2]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
 Session data awareness in practice: how engineers who understand the data lifecycle of their sessions compose prompts differently; the gap between assumed and actual session data handling.

[^3]: Anthropic — "Usage Policy and Enterprise Data Agreements," Anthropic Documentation, 2026. https://www.anthropic.com/legal/aup
 Enterprise vs. standard plan data handling: training opt-out, retention, and data residency provisions; the policy update cycle and the obligation to review current terms rather than cached assumptions.

[^4]: Stack Overflow — "Developers Remain Willing but Reluctant to Use AI: The 2025 Developer Survey Results," December 29, 2025. https://stackoverflow.blog/2025/12/29/developers-remain-willing-but-reluctant-to-use-ai-the-2025-developer-survey-results-are-here/
 Developer assumptions about session data privacy: the gap between what developers believe about AI session data handling and what current policies say.

[^7]: Judy Hanwen Shen and Alex Tamkin (Anthropic) — "How AI Assistance Impacts the Formation of Coding Skills," arXiv:2601.20245, January 28, 2026. https://arxiv.org/abs/2601.20245
 Data processing addenda under enterprise API agreements: the practical difference between standard and enterprise data handling terms for teams with compliance requirements.

[^9]: CIO — "How Agentic AI Will Reshape Engineering Workflows in 2026," April 2026. https://www.cio.com/article/4134741/how-agentic-ai-will-reshape-engineering-workflows-in-2026.html
 Annual data governance review practice: organizational processes for maintaining accurate understanding of AI tool data handling terms under evolving privacy obligations.

[^10]: DEV Community — "AI Is Creating a New Kind of Tech Debt — And Nobody Is Talking About It," March 2026. https://dev.to/harsh2644/ai-is-creating-a-new-kind-of-tech-debt-and-nobody-is-talking-about-it-3pm6
 Production data in debugging sessions: the mechanism by which real customer data enters AI sessions through routine debugging workflows; the inadvertent vs. deliberate distinction.

[^11]: Yonatan Sason — "The Black Box Problem: Why AI-Generated Code Stops Being Maintainable," *Towards Data Science*, March 6, 2026. https://towardsdatascience.com/the-black-box-problem-why-ai-generated-code-stops-being-maintainable/
 Data classification frameworks for AI session content: how to tier session content by sensitivity and apply different handling rules to each tier.

[^12]: Dex Horthy (YC Root Access) — "Advanced Context Engineering for Agents," YouTube, August 2025. https://www.youtube.com/watch?v=IS_y40zY-hc
 - CLAUDE.md session scope configuration: specific syntax for adding data handling reminders to module-level configuration files; the agentic session scope control mechanisms
 - Data minimization in practice: how to configure Claude Code sessions to limit context scope in sensitive modules while maintaining AI assistance effectiveness
 - Enterprise API configuration for data privacy: the settings that affect training data contribution and retention for teams with regulatory obligations

[^13]: Ravikanth Konda — "Human-AI Collaboration in Software Teams: Evaluating Productivity, Quality, and Knowledge Transfer with Agentic and LLM-Based Tools," *International Journal of AI, BigData, Computational and Management Studies*, February 17, 2026. https://ijaibdcms.org/index.php/ijaibdcms/article/view/418
 Competitive exposure through AI sessions: the risk profile for unreleased feature information, roadmap details, and proprietary design decisions included in AI development sessions.

[^15]: GitHub — "Octoverse 2025: The State of AI in Software Development," GitHub Octoverse 2025. https://github.blog/news-insights/research/the-state-of-open-source-and-ai/
 Team-level AI data governance: organizational practices for maintaining session data policies across a team using AI tools daily; the gap between initial adoption policies and current practice.

[^16]: Fannar Steinn Aðalsteinsson et al. — "Rethinking Code Review Workflows with LLM Assistance: An Empirical Study," arXiv:2505.16339, May 22, 2025. https://arxiv.org/abs/2505.16339
 CLAUDE.md as a data governance tool: how module-level session configuration files function as operational privacy reminders rather than policy documents; the specificity requirement for behavioral effectiveness.

[^a]: [Governance: AI Usage Policy](../Governance/02-ai-usage-policy.md) — privacy constraints on what data enters AI sessions are a core usage policy requirement; this ethical analysis is the basis for those policy provisions.

[^b]: [Security: Secrets Management](../Security/04-secrets-management.md) — secrets entering AI sessions are a training data privacy risk; the operational and ethical perspectives address the same exposure.

[^c]: [Tooling: Settings and Permissions](../Tooling & Configuration/05-settings-and-permissions.md) — permission scoping controls what Claude Code can access; privacy protection is implemented through permission boundaries.

[^d]: [Governance: Compliance and Audit](../Governance/06-compliance-and-audit.md) — data privacy compliance is a primary audit category; the ethical analysis defines what the compliance documentation must demonstrate.
