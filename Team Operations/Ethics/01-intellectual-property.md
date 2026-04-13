## Intellectual Property: Code Attribution and License Risk in AI-Assisted Development

**Related to:** [Ethics Overview](00-overview.md) — Risk 1 · [Governance: AI Usage Policy](../Governance/02-ai-usage-policy.md)[^a] · [Governance: Compliance and Audit](../Governance/06-compliance-and-audit.md)[^b] · [Security: Dependency Security](../Security/03-dependency-security.md)[^c]

---

## Overview

When an engineer writes code by hand, the intellectual property provenance is clear: the code is the engineer's expression of an idea, the company owns it as work product, and if it draws on open-source libraries, those dependencies are documented through standard package management. When AI generates code, the provenance is less clear. The model was trained on a large corpus that includes code under many different licenses; when it generates an implementation, there is a meaningful probability that the output is similar to or derived from specific training data sources in ways that may have IP implications.

This is not a hypothetical concern. The legal landscape in 2026 includes active litigation around AI training data and generated outputs. Several jurisdictions have issued preliminary guidance suggesting that AI-generated code can create license obligations when it is substantially similar to licensed training data.[^2] For a small team without legal counsel on staff, the practical question is not how to resolve these questions definitively — courts are still doing that — but how to reduce exposure through reasonable due diligence. This memo covers those practices.

---

## Section 1: Understanding the IP Risk Landscape

**Description:** AI models are trained on code from public repositories, including code under GPL, LGPL, AGPL, and other copyleft licenses that impose restrictions on derivative works. The training process is not copyright infringement in itself under current US doctrine (fair use arguments generally apply), but the generated outputs occupy more contested legal ground. A generated function that closely resembles a GPL-licensed implementation may carry GPL obligations even though the engineer did not directly copy it.[^3]

The risk is not uniform. For implementations of widely-used algorithms, common utility functions, or standard API patterns, the risk of close similarity to licensed training data is higher — because these patterns appear frequently in training data and the model may reproduce them faithfully. For custom business logic specific to the team's domain, the risk is lower — because this type of code is rare in training data and the model has less specific material to reproduce.[^4]

**Recommended Practice:**
- Develop a mental model of high-vs-low IP risk AI generation contexts: high risk (standard algorithm implementations, common utility functions, well-known API patterns from major libraries), low risk (business domain logic, team-specific architectural patterns, novel combinations of standard components).[^3]
- For high-risk contexts, run a brief similarity check before merge: GitHub Copilot provides duplicate detection inline; for non-Copilot contexts, searching for distinctive code fragments in public repositories is a manual but effective check.[^2]
- When generating implementations of specific algorithms (sorting, compression, cryptography), prefer to reference well-known public-domain descriptions and ask Claude to implement them from the specification rather than from a code example. "Implement Dijkstra's algorithm as described in [specification]" produces a cleaner IP provenance than "implement this graph traversal function."[^4]
- Keep a brief IP review log for any AI-generated implementation of a non-trivial algorithm or utility function: what was generated, whether a similarity check was run, the result, and the decision. This log is not legally required but provides a record of due diligence if questions arise later.

---

## Section 2: Open-Source License Compliance

**Description:** Teams using AI-generated code in commercially licensed products need to be particularly attentive to the presence of copyleft-licensed patterns in AI output. AGPL-licensed code in particular carries strong copyleft obligations that extend to network-accessible services; inadvertent inclusion of AGPL patterns in a SaaS product could create obligations to release the entire product's source code.

The practical challenge is that AI does not disclose the license status of patterns it reproduces. An engineer who directly copies code from GitHub would see the license file and evaluate it; an engineer who uses AI to generate functionally identical code receives no license disclosure. The due diligence obligation exists regardless — the mode of incorporation does not change the license status of the incorporated material.[^3]

**Recommended Practice:**
- Add a CLAUDE.md instruction for security-sensitive and algorithm-heavy code generation: "When implementing known algorithms or common utility functions, note any specific libraries or implementations you are drawing on. If the implementation closely resembles a specific open-source project, identify it." This prompts Claude to disclose the provenance of its outputs rather than treating it as implicit.[^6]
- Configure the team's dependency scanning tool (Snyk, Dependabot, or equivalent) to flag any packages added to `package.json` or equivalent that carry copyleft licenses requiring commercial license review. AI-generated code that suggests adding new dependencies is a specific IP risk surface that automated tooling can address.
- When a similarity check reveals close similarity to a GPL or AGPL-licensed implementation, escalate to the architect for IP review before merge. Do not proceed under time pressure — an inadvertent AGPL inclusion is a more expensive problem to unwind than a delayed feature.[^2]
- Document the team's standard acceptable licenses in CLAUDE.md: "Dependencies and implementations should use MIT, Apache 2.0, BSD, or ISC licenses. Do not suggest GPL or AGPL-licensed alternatives." This creates a standing instruction that filters AI suggestions at the generation stage rather than the review stage.[^4]

---

## Section 3: Code Ownership and Attribution

**Description:** AI-generated code raises a subtler ownership question beyond license compliance: who owns the code, and who is responsible for it? In most jurisdictions, work product created by employees using company tools is owned by the employer. AI-generated code produced during employment generally falls within this framework. But the question of who is *responsible* for AI-generated code — who can explain it, maintain it, and be held accountable for its quality — is less cleanly answered by employment law and more dependent on team governance practices.

The comprehension debt problem (see Issues — Comprehension Debt) is directly connected to code ownership: code that nobody understands is code that nobody effectively owns. When a defect in AI-generated code causes a production incident, the question of who was responsible for it will focus on who reviewed and approved it, not who (or what) generated it. Review and approval are the ownership mechanisms that legal and organizational accountability frameworks recognize.[^8]

**Recommended Practice:**
- Require explicit author attribution in all PRs: the PR author is the accountable person for the AI-generated code in that PR, not the AI tool. This framing — which is legally correct — should be explicit in team culture rather than implicit.
- Apply the "can you explain it" review rule specifically as an accountability mechanism: if the PR author cannot explain the AI-generated code, they cannot own it in any meaningful sense. The rule is not punitive — it is the mechanism by which the PR author takes genuine rather than nominal ownership of what they are shipping.[^8]
- Include a "reviewed AI-generated code" attestation in the PR merge criteria for AI-primary PRs: the PR author attests that they have read, understood, and taken ownership of the AI-generated content. This creates an explicit record of the ownership transfer that occurs at review.[^9]
- Brief the team on the ownership framework: "Claude generates the code; we own it. That means we are responsible for understanding it, maintaining it, and being accountable for its quality." This framing matters for culture — engineers who understand they own AI-generated code approach review differently than engineers who see AI as a separate author.

---

## Section 4: Training Data Concerns

**Description:** When engineers use Claude Code, the sessions may generate data that contributes to future model training depending on Anthropic's data use practices and the team's API terms. This creates a potential feedback loop: AI-generated code from the team's sessions might eventually appear in training data that influences future model outputs. For most teams, this is a background concern rather than an active risk — but for teams working on genuinely novel algorithms, proprietary business logic, or unreleased product features, it warrants consideration.[^10]

Anthropic's data retention and training policies are documented and have been updated as the regulatory environment has evolved. The current policies should be reviewed annually and whenever the team's compliance obligations change — what was acceptable under a previous policy revision may require reevaluation after an update.[^10]

**Recommended Practice:**
- Review Anthropic's current data use policy annually and confirm that the team's API terms reflect the team's current data governance requirements. Enterprise plans typically include stronger data retention and training opt-out provisions than individual plans.[^10]
- For sessions involving genuinely proprietary or pre-release material, use an enterprise API plan with appropriate data use terms rather than assuming default terms are adequate. The cost difference between plans is typically low relative to the IP value of what is being worked on.[^11]
- Add a CLAUDE.md note for sessions involving upcoming product features or unreleased integrations: "This session involves pre-release features. Do not include identifying product or customer information in prompts." This is a session-scope reminder rather than a comprehensive privacy control — it reduces the probability of inadvertent disclosure without guaranteeing it.[^6]
- Brief the team on Anthropic's data practices during AI workflow onboarding. Engineers who understand how their session data is used make more deliberate decisions about what to include in sessions than those who have never thought about it.[^9]

---

## Section 5: Practical IP Review Protocol

**Description:** A practical IP review protocol for a small team does not require external legal counsel for routine AI-assisted development. It requires a lightweight checklist applied to AI-generated PRs involving non-trivial implementations, escalation criteria for situations that exceed the checklist's scope, and a standing relationship with legal counsel for escalated cases.[^2]

The checklist is not overhead — it is the due diligence record that protects the team if IP questions are raised in the future. A team that reviewed a PR and found no concerns has a better posture than a team that merged without review, even if the outcome (no IP issue) is the same in both cases.[^3]

**Recommended Practice:**
- Create a two-minute IP review checklist for AI-primary PRs: (1) Does this implement a known algorithm or common utility pattern? (2) Does any generated code look familiar as a specific open-source implementation? (3) Does any generated code suggest adding new dependencies with non-permissive licenses? (4) Does this involve proprietary business logic that should not be in session data? Record the checklist responses in the PR.[^2]
- Define escalation criteria: escalate to the architect if the answer to questions 1 or 2 is "yes and I recognize it as a specific project." Escalate to legal counsel if the architect believes the similarity is close enough to require formal IP analysis.[^3]
- Build a brief IP case library: when an IP review finds close similarity and a decision is made, document it (what was found, what was decided, why). This case library accelerates future reviews by providing precedent for similar cases and provides evidence of due diligence for any future audit.
- Run the IP review checklist for all new MCP server integrations as well as for code generation. MCP servers that fetch and incorporate external code, templates, or snippets into AI-generated output are a less obvious IP risk surface that the code generation checklist does not cover.[^4]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| IP Risk Landscape | Brief team on high vs. low IP risk generation contexts | Architect |
| License Compliance | Add acceptable licenses list to CLAUDE.md; configure dependency scanning | Backend lead |
| Code Ownership | Add AI-generated code attestation to PR merge criteria | Architect |
| Training Data | Review Anthropic data use policy; confirm enterprise API terms | CTO |
| IP Review Protocol | Create two-minute IP checklist; define escalation criteria | Architect + CTO |

---

[^2]: daily.dev — "Vibe Coding in 2026: How AI Is Changing the Way Developers Write Code," April 2026. https://daily.dev/blog/vibe-coding-how-ai-changing-developers-code
 Similarity check as due diligence practice: how to identify AI-generated code that closely resembles specific licensed implementations before merge.

[^3]: DEV Community — "AI Is Creating a New Kind of Tech Debt — And Nobody Is Talking About It," March 2026. https://dev.to/harsh2644/ai-is-creating-a-new-kind-of-tech-debt-and-nobody-is-talking-about-it-3pm6
 IP risk profile by implementation type: high-risk contexts (standard algorithms, common utilities) vs. low-risk contexts (business domain logic, novel combinations).

[^4]: Vocal/Futurism — "8 AI Code Generation Mistakes Devs Must Fix to Win 2026." https://vocal.media/futurism/8-ai-code-generation-mistakes-devs-must-fix-to-win-2026
 Acceptable license list in CLAUDE.md: using system prompt constraints to filter AI suggestions at generation rather than review; specification-based vs. example-based algorithm implementation.

[^6]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
 CLAUDE.md IP and data governance instructions: provenance disclosure prompts, data sensitivity reminders, and session-scope constraints for proprietary material.

[^8]: CodeRabbit — "State of AI Code Generation: AI vs. Human Code Report," December 17, 2025. https://www.coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report
 Review and approval as ownership mechanisms: the accountability framework that makes the PR reviewer the responsible party for AI-generated code regardless of AI authorship.

[^9]: Fannar Steinn Aðalsteinsson et al. — "Rethinking Code Review Workflows with LLM Assistance: An Empirical Study," arXiv:2505.16339, May 22, 2025. https://arxiv.org/abs/2505.16339
 AI code review attestation: how explicit ownership statements in PR process create the accountability record that legal and organizational frameworks need.

[^10]: Anthropic — "Privacy and Data Handling," Claude Code Documentation, 2026. https://code.claude.com/docs/en/privacy-data-handling
 Data use policy for Claude Code sessions: training data practices, retention terms, enterprise plan provisions, and the annual review obligation for teams with compliance requirements.

[^11]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
 Enterprise API plan considerations: the data governance differences between individual and enterprise plans relevant to teams working with proprietary material.

[^13]: Dex Horthy (YC Root Access) — "Advanced Context Engineering for Agents," YouTube, August 2025. https://www.youtube.com/watch?v=IS_y40zY-hc
 - CLAUDE.md for IP governance: using the system prompt to embed IP sensitivity markers, acceptable license constraints, and data handling reminders at the session level
 - Session data practices: how to configure sessions involving proprietary material to minimize inadvertent disclosure while maintaining AI assistance effectiveness
 - Enterprise API configuration: the specific settings that affect data use and training data contribution for teams with IP sensitivity requirements

[^a]: [Governance: AI Usage Policy](../Governance/02-ai-usage-policy.md) — usage policy operationalizes the IP risk analysis here into prohibited and permitted use categories; the ethical analysis and the policy are paired.

[^b]: [Governance: Compliance and Audit](../Governance/06-compliance-and-audit.md) — IP compliance is a recurring audit requirement; the ethical analysis determines what evidence the audit must capture.

[^c]: [Security: Dependency Security](../Security/03-dependency-security.md) — dependency license selection is a primary IP risk vector; AI-generated dependency choices require the same license review the ethical analysis demands.
