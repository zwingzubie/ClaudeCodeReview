## Overview

For a team of 11 — 4 backend engineers, 3 frontend engineers, 1 architect, 1 QA engineer, and 2 product managers — with a former CTO closely involved in velocity metrics, the proximity of leadership to engineering creates a specific governance risk: there is no structural buffer between "we can ship faster with AI" and "let's ship faster with AI." Good governance is not about slowing the team down. It is about ensuring that the speed AI provides does not outpace the team's ability to maintain quality, security, and architectural coherence — because in the absence of deliberate governance, it reliably will.[^1]

Veracode's Spring 2026 analysis found that 45% of AI-generated code fails security tests — a rate unchanged despite major capability improvements in underlying models.[^2] Anthropic's 2026 Agentic Coding Trends Report found that developers can fully delegate only 0–20% of tasks, with 80–100% requiring active human supervision and judgment.[^3] These findings, taken together, describe a governance gap: AI increases output velocity materially while requiring ongoing human judgment at a high rate — and judgment requires structures that ensure it is actually applied rather than assumed. Without those structures, velocity accumulates debt in forms that compound: security vulnerabilities, architectural drift, and comprehension gaps that are invisible in CI dashboards and only surface as incidents.

Six governance policy areas are documented below. They are ordered by immediacy: PR review policies protect every merge; acceptable use guidelines define team-wide standards; sprint planning gates prevent oversized AI commitments before they accumulate; escalation procedures define what happens when standards are violated; quarterly reviews provide oversight at the leadership level; and compliance requirements protect the organization in regulated contexts.

---

## Policy 1: AI Code Review Policy

**Description:** Standard code review practices were designed for human-authored code. They assume that the reviewer can engage with the code as a representation of the author's reasoning — and that asking the author about their decisions will produce informative answers. AI-generated code breaks both assumptions: the "reasoning" behind it is statistical inference rather than deliberate design, and the engineer who submitted the PR may not be able to explain it in meaningful depth. A review process that does not adapt to these differences will catch fewer issues in AI-generated code, not more — while consuming the same reviewer time.[^4]

A 2026 analysis found that AI-generated PRs had logic and correctness issues 75% more common and readability problems 3× more prevalent than human-written PRs.[^5] Reviewers who treat AI-generated code with the same review posture as human-written code — looking primarily for syntax errors and logic bugs — miss the structural and architectural issues that are AI-generated code's distinctive failure mode. Review policies that account for these differences produce materially better outcomes.[^6]

**Proposed Solution:**
- Establish a PR size limit for AI-generated code: 300–400 lines maximum per PR. Larger outputs must be split before review. This is not arbitrary; research consistently shows that review quality degrades significantly above 400 lines, and AI-generated PRs tend to be larger than human-authored ones.[^7]
- Separate architectural review (does this fit the system?) from implementation review (does this work correctly?). These require different reviewers operating in different cognitive modes. The architect reviews for the former; implementation engineers for the latter. AI makes implementation quality harder to distinguish from architectural soundness — separating them is the countermeasure.[^6]
- Apply the "can you explain it" rule to all AI-primary PRs: the author must be able to walk through the logic, the tradeoffs, and the alternatives considered. A PR whose author cannot explain it does not pass architectural review regardless of test status.[^8]
- Run a fresh-context reviewer session (Workflow 6: Writer/Reviewer Pattern) before requesting human review for non-trivial AI-primary PRs. The AI reviewer session catches the class of issues a human reviewer would need to invest significant time to find — giving human reviewers a preprocessed finding set to evaluate rather than a blank slate.[^6]

---

## Policy 2: Acceptable Use Guidelines

**Description:** Eight engineers using Claude Code in eight different ways generates not just inconsistent outputs but inconsistent risks. An engineer who passes production PII to a session, an engineer who uses a session to make unreviewed configuration changes to a shared environment, and an engineer who uses agentic mode without a human review step before merge all create risks that compound when they occur simultaneously or go undetected. Acceptable use guidelines define the boundaries of authorized Claude Code use — not to restrict legitimate productivity, but to ensure that high-risk uses are explicit and reviewed rather than accidental and invisible.[^9]

The lack of shared standards is a team vulnerability. The Pragmatic Engineer's 2026 AI tooling survey found that at companies with under 20 engineers — our team's size — AI acceptable use policies were present in only 23% of organizations, despite widespread individual AI tool adoption. The absence of a policy is not neutral: it means high-risk uses are possible without any governance mechanism to surface or address them.

**Proposed Solution:**
- Define authorized Claude Code usage categories: AI-assisted (human writes, AI augments), AI-primary (AI generates first pass, human reviews before merge), and agentic-delegated (AI runs unattended, output reviewed before any production effect). Each category has different review requirements — document them in the team's engineering handbook.[^3]
- Define prohibited uses explicitly: no production PII in sessions, no unreviewed agentic execution affecting shared environments, no `--dangerouslySkipPermissions` flag outside of explicitly approved use cases documented in advance.[^9]
- Require the architect's approval for first-time use of any new Claude Code capability (new MCP server types, new permission profiles, new agentic delegation patterns). This is not a bureaucratic gate — it is a two-minute conversation that ensures new capabilities are introduced with deliberate consideration rather than casual experimentation.[^11]
- Review the acceptable use guidelines annually and whenever Anthropic releases significant new Claude Code capabilities. A guideline written for today's tool configuration may not be appropriate for next year's tool.[^9]

---

## Policy 3: Sprint Planning and AI Readiness Gates

**Description:** AI is not equally effective across all task types. The METR productivity studies found a 19% slowdown on complex tasks where AI was expected to accelerate work.[^12] The Anthropic Agentic Trends Report found that only 0–20% of tasks are fully delegatable without ongoing supervision.[^3] Sprint planning that does not account for these limitations — that classifies all tasks as equally AI-suitable — tends to produce sprints with unrealistic velocity expectations, incomplete stories when AI fails on complex tasks, and accumulating comprehension debt when AI-primary completion rates mask human understanding deficits.[^13]

AI readiness assessment is a sprint planning skill distinct from traditional story pointing. It requires evaluating tasks on three dimensions before assignment: how well-specified is the task (fully specified tasks are AI-suitable; ambiguous ones are not), how high-precedent is the pattern in the codebase (tasks that mirror many existing implementations are AI-suitable; novel patterns are not), and how critical is the correctness (security-critical and architecture-defining tasks warrant higher human involvement regardless of apparent AI suitability).[^13]

**Proposed Solution:**
- Add an AI readiness classification to sprint planning: each task is classified as AI-primary (suitable for AI-first execution), AI-assisted (suitable for AI support with human lead), or human-primary (requires human-first execution). This classification does not change the story point estimate but sets expectations about the workflow.[^12]
- Establish a sprint-level AI code percentage target: AI-primary tasks should not exceed 40% of sprint capacity. Above this threshold, rework rates increase materially — making the sprint appear high-velocity while accumulating the debt that slows subsequent sprints.[^14]
- For tasks classified as AI-primary, require a spec.md before sprint execution begins. A task that cannot be specified in a spec.md is likely not AI-suitable — the specification difficulty is a signal about task clarity, not just documentation overhead.[^13]
- After each sprint, review the AI classification accuracy: which tasks classified as AI-primary required significant human correction? Which tasks classified as human-primary could have benefited from more AI assistance? Use this data to improve classification accuracy over time.[^12]

---

## Policy 4: Escalation and Override Procedures

**Description:** Governance policies are only effective if there are clear procedures for what happens when they are violated, when engineers encounter edge cases not covered by policy, and when the policy itself is incorrect and needs to change. Without escalation procedures, policy violations either go unaddressed (creating the impression that policies are aspirational rather than operational) or are handled ad hoc in ways that are inconsistent, create precedent, and generate team tension.[^15]

Override procedures matter equally: policies designed for 80% of cases may be wrong for specific situations. An engineer who needs to use agentic mode for a legitimate but policy-exceeding use case should have a documented path to get temporary authorization rather than either breaking policy silently or abandoning the legitimate need. A governance structure without override procedures creates pressure toward silent violations.[^11]

**Proposed Solution:**
- Define escalation paths for three classes of situation: policy violation (an engineer did something outside authorized use), edge case (a situation the policy doesn't clearly address), and policy error (the policy is wrong for a legitimate use case). Each has a different escalation owner and resolution process.[^15]
- Policy violations: escalate to architect for immediate review. First occurrence is a coaching conversation; recurrent violations are escalated to CTO. Document the incident and the resolution, regardless of severity, so patterns are visible across time rather than only within individual conversations.[^15]
- Override requests: engineer documents the use case and the policy it would exceed; architect reviews and either approves with conditions or denies with alternative recommendation; decision is logged. If multiple similar overrides are approved, the policy should be updated to accommodate the pattern.[^11]
- Policy errors: any engineer can propose a policy change via the standard PR process against the governance documentation. The architect reviews; the CTO approves changes with broader team impact. Governance documentation treated like production code — versioned, reviewed, and changed deliberately.[^1]

---

## Policy 5: Quarterly Engineering Health Review

**Description:** Sprint-level and monthly metrics (see Metrics section) provide operational visibility. The quarterly engineering health review provides the strategic oversight layer: a structured conversation between the architect and the CTO that evaluates whether the team's AI governance practices are producing the intended outcomes, whether the policies established at the beginning of the quarter remain appropriate, and whether emerging risks require new governance attention. This is the session where rework trends, security findings, and codebase health indicators are interpreted as leadership-level signals rather than operational details.[^16]

Google's DORA 2026 research found a 7.2% decrease in delivery stability correlated with increased AI adoption — a signal that appears in aggregated data but is invisible to individual sprint reviews.[^2] The quarterly health review is the cadence at which this kind of aggregate signal becomes visible and actionable at the leadership level rather than diffusing into the noise of weekly status updates.

**Proposed Solution:**
- Structure the quarterly engineering health review as a 60-minute architect-to-CTO presentation: AI code volume trend, rework rate trend, security vulnerability trend, codebase health indicators (size, duplication, complexity, test coverage), CLAUDE.md currency, and policy compliance summary.[^16]
- The CTO's explicit responsibility in this review is to evaluate whether velocity metrics are being pursued at the cost of quality and sustainability. If rework is rising, codebase complexity is increasing, and security findings are accumulating, the velocity number requires reinterpretation — it is measuring output while obscuring debt.[^1]
- At each quarterly review, evaluate whether any existing governance policies should be updated, deprecated, or extended. AI tooling evolves faster than annual policy review cycles; quarterly review keeps the governance framework current with the tool's capabilities.[^11]
- Document decisions made in the quarterly review: what was observed, what it was interpreted as signaling, and what governance changes were made in response. This creates an audit trail of governance decisions that is valuable if questions arise later about why the team made specific choices at specific times.[^15]

---

## Policy 6: Compliance and Audit Requirements

**Description:** For teams with enterprise customers, regulated-industry clients, or contractual security obligations, AI-generated code creates new compliance requirements that extend beyond standard engineering governance. SOC 2 Type II audits increasingly ask about AI tool usage and the controls around it; security questionnaires from enterprise prospects now routinely include questions about AI code generation practices; and data processing agreements may be affected by what data enters AI model sessions. A team that has not addressed these requirements is not prepared for the audit cycle they will face as the company scales.

Compliance requirements around AI are evolving rapidly. What was not asked in a 2024 SOC 2 audit may be a required control in a 2026 audit. The teams that manage this transition best are those that have proactively documented their AI governance practices in audit-ready form — not as a compliance exercise, but as a natural output of the governance work described in this section.

**Proposed Solution:**
- Maintain an AI governance log as a standard engineering artifact: what AI tools are in use, what policies govern their use, who owns each policy, and when each policy was last reviewed. This is the primary artifact for AI-related compliance inquiries.
- Review Anthropic's SOC 2 Type II attestation and any applicable BAAs (Business Associate Agreements for HIPAA contexts) and include these in the team's vendor documentation alongside other critical infrastructure providers.[^12]
- When responding to customer security questionnaires or RFP security sections, have a prepared statement about AI code generation practices: the tools used, the governance policies in place, and the security controls applied to AI-generated code. This should be reviewed and updated annually.
- As AI-specific regulations emerge (the EU AI Act's implications for code generation tools are still being interpreted as of Q1 2026), assign the CTO responsibility for monitoring regulatory developments and triggering governance reviews when new requirements apply to the team's practices.[^9]

---

---

## Scaling Beyond 11 Engineers

The governance policies documented here were designed for a team of 11. Not all of them scale linearly as the team grows. When headcount reaches 15 or above, three structural changes become necessary before the current policies break down.

First, the architect can no longer be the sole reviewer for CLAUDE.md corrections, override requests, and new capability authorizations without creating a blocking bottleneck. At 15+ engineers, assign domain leads (backend lead, frontend lead) as co-reviewers for the instruction files they own, with the architect retaining final authority on the root CLAUDE.md and security-invariants files. The CODEOWNERS file should reflect this tiered ownership explicitly.[^11]

Second, the agentic-delegated usage category requires more formal governance at scale. With 11 engineers, informal architect notification before sessions exceeding two hours is workable. With 15 or more, a lightweight request-and-approval workflow (a two-field form: task description and checkpoint plan) replaces informal notification, ensuring the architect has documented context for every authorized agentic session rather than relying on recall.[^3]

Third, the quarterly engineering health review shifts from an architect-to-CTO presentation to a cross-functional review including domain leads. The dataset grows too large for one person to interpret without domain context; domain leads provide the contextual interpretation that keeps aggregate metrics actionable. The CTO's role remains strategic oversight rather than operational management of the specifics.[^16]

These changes are not improvements to the current governance — they are anticipatory adaptations for a scale at which the current structure would produce bottlenecks or lose coverage. Review whether any of these structural changes are warranted at each quarterly health review, using team size and policy violation frequency as the primary indicators.

---

## Summary of Recommended Actions

| Policy Area | Immediate Action | Owner |
|---|---|---|
| AI Code Review Policy | Set 300–400 line PR limit; add "can you explain it" rule to PR template | Architect |
| Acceptable Use Guidelines | Define three usage categories and prohibited uses; document in handbook | Architect + CTO |
| Sprint Planning Gates | Add AI readiness classification to sprint planning; set 40% cap | Architect |
| Escalation and Overrides | Define three escalation paths; establish override documentation process | Architect |
| Quarterly Health Review | Schedule quarterly architect-to-CTO review; define presentation structure | Architect + CTO |
| Compliance and Audit | Create AI governance log; review vendor documentation | CTO |

---

[^1]: DEV Community — "AI Is Creating a New Kind of Tech Debt — And Nobody Is Talking About It," March 2026. https://dev.to/harsh2644/ai-is-creating-a-new-kind-of-tech-debt-and-nobody-is-talking-about-it-3pm6
 Governance as velocity management: the argument that deliberate governance enables sustainable velocity rather than limiting it; the structural pressure toward velocity-over-quality in CTO-adjacent teams.

[^2]: Veracode — "Spring 2026 GenAI Code Security Update: Despite Claims, AI Models Are Still Failing Security," March 24, 2026. https://www.veracode.com/blog/spring-2026-genai-code-security/
 45% AI code security failure rate; stagnation despite model improvements as evidence that governance rather than model capability is the primary lever for AI security outcomes.

[^3]: Anthropic — "2026 Agentic Coding Trends Report," Anthropic, 2026. https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf
 Human delegation ceiling: 0–20% fully delegatable tasks; 80–100% requiring active supervision. The foundational data point for acceptable use policy and sprint planning gate design.

[^4]: Fannar Steinn Aðalsteinsson et al. — "Rethinking Code Review Workflows with LLM Assistance: An Empirical Study," arXiv:2505.16339, May 22, 2025. https://arxiv.org/abs/2505.16339
 Standard review practices and their limitations for AI-generated code: cognitive load increase, reviewer unfamiliarity with AI inference patterns, and the structural differences between evaluating AI-generated vs. human-authored code.

[^5]: CodeRabbit — "State of AI Code Generation: AI vs. Human Code Report," December 17, 2025. https://www.coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report
 AI PRs with 75% more logic/correctness issues and 3× readability problems; the case for AI-specific review policies calibrated to AI's distinctive failure modes.

[^6]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
 Writer/reviewer pattern as a preprocessing step for human review: fresh-context review sessions that give human reviewers a structured finding set rather than a blank slate.

[^7]: Graphite — "Best Practices for Managing Pull Request Size." https://graphite.com/guides/best-practices-managing-pr-size
 Research supporting the 300–400 line PR limit: review quality, defect detection rates, and review time as a function of PR size; the specific decline threshold above 400 lines.

[^8]: daily.dev — "Vibe Coding in 2026: How AI Is Changing the Way Developers Write Code," April 2026. https://daily.dev/blog/vibe-coding-how-ai-changing-developers-code
 The "can you explain it" standard as an architectural review gate: why test passage is insufficient evidence of comprehension and why PR review must include explanation capability.

[^9]: Anthropic — "Security and Permissions," Claude Code Documentation, 2026. https://code.claude.com/docs/en/security-permissions
 Permission model documentation: authorized vs. prohibited operations, `--dangerouslySkipPermissions` flag documentation and risks, and the permission scoping framework for different session types.

[^11]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
 Override procedures and permission scope: when to escalate new capability requests; the discipline of treating governance documentation like production code.

[^12]: METR — "We Are Changing Our Developer Productivity Experiment Design," METR Research, February 2026. https://www.metr.org/blog/2026-02-24-uplift-update/
 19% productivity slowdown on complex tasks; AI readiness classification as a sprint planning practice; the evidence base for human-primary classification thresholds.

[^13]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
 Spec.md as sprint planning prerequisite: task specification as a readiness gate; "waterfall in 15 minutes" as the pre-sprint specification discipline.

[^14]: Kyros — "The Vibe Coding Crisis: How AI-Generated Technical Debt Is Costing Companies Millions," March 2026. https://usekyros.ai/blog/vibe-coding-crisis-ai-technical-debt
 40% AI code percentage threshold: above this level, rework rates increase materially; sprint-level monitoring as a governance intervention before accumulation becomes critical.

[^15]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
 Escalation and audit trail discipline: documenting incidents, override decisions, and governance changes for compliance and pattern recognition purposes.

[^16]: CIO — "How Agentic AI Will Reshape Engineering Workflows in 2026," April 2026. https://www.cio.com/article/4134741/how-agentic-ai-will-reshape-engineering-workflows-in-2026.html
 Quarterly engineering health review as a leadership governance mechanism: the "Delegate, Review, Own" framework applied at the organizational level; how delivery stability metrics require quarterly cadence to interpret.

[^18]: Dex Horthy (YC Root Access) — "Advanced Context Engineering for Agents," YouTube, August 2025. https://www.youtube.com/watch?v=IS_y40zY-hc
 - Governance through context: how team-level CLAUDE.md functions as a governance mechanism by encoding team standards into every session rather than relying on individual engineer compliance
 - Permission scoping: how task-level permission management reduces governance violations by making high-risk operations require explicit approval rather than passive acceptance
 - Audit infrastructure: how session logs and structured context provide the traceability that compliance requirements need

[^19]: Sabrina Ramonov — "CLAUDE CODE FULL COURSE," YouTube, February 17, 2025. https://www.youtube.com/watch?v=fYX6hHC9FhQ
 - Acceptable use in practice: how to configure Claude Code to enforce team policies through settings.json and CLAUDE.md rather than relying on engineer memory
 - Sprint planning integration: using task classification and spec.md requirements as workflow prerequisites that implement AI readiness gates in daily practice
 - Compliance-ready documentation: how the tutorial's CLAUDE.md and settings.json structure creates the audit trail that governance and compliance requirements need

