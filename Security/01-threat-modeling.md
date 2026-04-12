## Threat Modeling for AI-Assisted Development

**Related to:** [Security Overview](00-overview.md) — Security Area 1

---

## Overview

Threat modeling in a traditional engineering context is primarily an application exercise: identify the system's assets, map the attack vectors, document the mitigations. That exercise remains necessary — but AI-assisted development adds a second threat surface that most team threat models do not yet include: the development process itself. Claude Code reads files, executes commands, writes to the filesystem, calls external tools, and maintains session state across a working session. Each of these capabilities is a potential attack or failure vector, and a team that models application threats but not development-process threats has a systematic blind spot.[^1]

The scale of this blind spot is growing. Sonar's 2026 research found that AI now generates 42% of code in AI-adopting repositories. When AI writes nearly half of all code, the security properties of the AI development process are not a niche concern — they are a primary engineering security concern. Prompt injection attacks, where malicious content in repository files manipulates Claude Code's behavior during a session, have moved from theoretical to documented in 2025–2026 security research. Agentic sessions with broad file-write permissions and minimal human oversight amplify the impact of any such manipulation.[^2]

STRIDE — the threat modeling framework that categorizes threats as Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, and Elevation of Privilege — maps cleanly onto the threat categories introduced by AI development sessions. Applying STRIDE to Claude Code sessions produces actionable mitigations that belong in CLAUDE.md, session configuration, and the team's development workflow. The cadence for this threat modeling is not annual — on a team with AI-heavy sprints, it should occur whenever a significant new AI capability is introduced or a new class of sensitive asset enters the repository.[^3]

---

## Section 1: How AI-Generated Code Changes the Threat Surface

**Description:** Human-written code has a known failure mode distribution: logic errors, boundary condition mistakes, and the occasional security shortcut under time pressure. AI-generated code inherits these failure modes but adds a structurally different category: vulnerability patterns that appear correct, pass tests, and match common code conventions — but encode security weaknesses that are subtler than anything a human reviewer expects to find in competently written code. Sonar's 2026 analysis found that AI-generated code introduces vulnerabilities at 2.74 times the rate of human-written code, with a particular concentration in input handling, authentication logic, and cryptographic implementation.[^4]

The mechanism is not that AI is careless — it is that AI generates code that is statistically representative of its training data, and training data contains vulnerable code. When the majority of code on GitHub implementing a particular authentication pattern contains a subtle weakness, AI will reproduce that weakness. The team is not only responsible for the vulnerabilities it introduces deliberately or through oversight; it is now exposed to the aggregated security decisions of every developer who wrote similar code before the model's training cutoff. That expanded exposure is the primary change to the team's application threat surface under AI adoption.

**Recommended Practice:**
- Maintain an explicit list of AI-specific vulnerability patterns observed in the team's own code reviews and in external research (OWASP, Snyk, Veracode publications). Update this list quarterly and incorporate it into SAST rule configurations and the security review checklist.[^4]
- Treat AI-generated code in security-critical paths — authentication, authorization, input validation, cryptographic operations, external API integration — as requiring security-specific review rather than functional review only. The vulnerability concentration in these categories is documented; the review calibration should reflect it.[^5]
- When a vulnerability is found in AI-generated code, add the pattern to the team's CLAUDE.md as a negative constraint: "Do not use pattern X in authentication handlers." This converts the post-incident lesson into a pre-session constraint that prevents recurrence.[^6]
- Track the team's AI-generated vulnerability rate as a security metric (see Metrics — Security Vulnerability Trends). Visibility into this rate is the feedback loop that tells the team whether its security controls are calibrated correctly to the actual AI vulnerability output.

---

## Section 2: STRIDE Applied to AI Development Sessions

**Description:** STRIDE provides a structured vocabulary for the threats that AI development sessions introduce. Spoofing: prompt injection in repository files, documentation, or test fixtures can manipulate Claude Code's behavior to produce outputs the engineer did not intend — the AI session is "spoofed" into executing a different task than the engineer requested. Tampering: AI-generated code that modifies security-critical logic in ways that are not obviously wrong can introduce backdoors or weakened controls that survive review. Information Disclosure: sessions that read broadly from the codebase may incorporate sensitive values into generated outputs or logs. Elevation of Privilege: agentic sessions run with file-write or command-execute permissions broader than the task requires can make modifications with higher impact than the engineer intended to authorize.[^3]

These are not equivalent in likelihood or impact for the team's current configuration. Prompt injection and information disclosure are the highest-probability categories: the team's repositories contain files that could plausibly carry injection payloads (test fixtures, data files, external content), and Claude Code's broad file-reading behavior means sensitive content in scope files can appear in session context. Tampering and privilege escalation are lower probability in interactive sessions but become significant concerns in agentic or headless configurations.[^1]

**Recommended Practice:**
- Document a STRIDE analysis for Claude Code in the team's security documentation, mapping each threat category to specific mitigations: prompt injection mitigations (`.claudeignore`, constrained session scope), information disclosure mitigations (`.claudeignore`, sensitive asset exclusion), and privilege escalation mitigations (minimum-permission session profiles).[^3]
- Configure Claude Code sessions with the minimum permission scope required for the task. A session writing unit tests does not need file-system access to the `secrets/` directory or command-execute access to deployment scripts. Use CLAUDE.md and session configuration to constrain scope proactively.[^6]
- Treat untrusted content (content fetched from the web, content generated by external systems, test fixtures containing realistic-looking data) as potential prompt injection vectors. Do not pass untrusted content to Claude Code sessions without reviewing it first.[^2]
- For agentic sessions with elevated permissions, apply the "minimal footprint" principle: use the most restrictive permission profile that allows the session to accomplish its task. Document the permission rationale for any session using command-execute or broad file-write access.[^1]

---

## Section 3: Threat Modeling Cadence for AI-Heavy Sprints

**Description:** Threat modeling as a one-time exercise is insufficient for an AI-assisted development workflow where the capabilities used, the assets in scope, and the team's risk tolerance evolve sprint to sprint. A session that was appropriate last month — agentic refactoring of a module that contained no sensitive logic — may be inappropriate this month if that module now handles authentication. The team needs a cadence that revisits the threat model at the frequency at which the team's AI capability usage and sensitive asset distribution changes.[^5]

The practical format for sprint-level threat modeling is not a full STRIDE analysis — it is a lightweight review that answers three questions before AI-heavy sprint work begins: Are we using any Claude Code capabilities in this sprint that we have not used before? Are any of the components we plan to have AI generate in security-critical paths? Have any sensitive assets (credentials, PII, proprietary business logic) moved into scope for planned sessions? If all three answers are no, the existing threat model covers the sprint. If any answer is yes, a brief threat model update is warranted before the sprint begins.

**Recommended Practice:**
- Assign the architect as threat model owner for AI-assisted development. The architect reviews planned sprint AI usage against the existing threat model and flags cases where an update is needed before sprint execution begins.[^5]
- Maintain a living threat model document (not a slide deck that is never updated) that records current Claude Code capability usage, sensitive assets in scope for AI sessions, active mitigations, and residual risk acceptance decisions. Review and update this document at the start of each sprint planning cycle.[^3]
- When a new Claude Code capability is introduced — a new MCP server type, a new permission profile, a new agentic workflow pattern — treat it as a threat model trigger. The capability introduction event, not the sprint boundary, is the appropriate review point for new capabilities.[^6]
- Document threat model decisions, including decisions that no update is needed. A record that shows the team reviewed the threat model and found no change required is evidence of a functioning process, not a gap in documentation.

---

## Section 4: Integrating Threat Models with CLAUDE.md Constraints

**Description:** A threat model that produces a document no one reads during a development session has limited operational value. The mechanism through which threat model outcomes affect session behavior is CLAUDE.md: constraints encoded there are present at the start of every session, applied without requiring engineers to remember what the threat model concluded about a given capability or file. The connection between threat modeling and CLAUDE.md is the operational bridge that gives threat modeling practical effect in a fast-moving AI-assisted workflow.[^6]

The mapping is direct: every threat model mitigation that takes the form "Claude Code should not do X" or "Claude Code should not read Y" or "Claude Code should use pattern Z instead of pattern A" should be encoded as a CLAUDE.md constraint. Mitigations that take the form "engineers should do X before running a session" should be encoded as workflow documentation but also as CLAUDE.md reminders that surface the requirement at the session level. The residual risk of a mitigation that depends on engineers remembering to do something is always higher than the residual risk of a mitigation encoded in session configuration.[^1]

**Recommended Practice:**
- After each threat model review, produce a CLAUDE.md diff as the primary output artifact. Threat model sessions that produce only a document and not a configuration change have not completed their operational objective.[^6]
- Structure CLAUDE.md security constraints in a dedicated section (e.g., `## Security Constraints`) with subsections for file access restrictions, code generation prohibitions, and required patterns. This organization makes the security constraints visible and maintainable rather than scattered through a general configuration file.[^6]
- Include threat model rationale as CLAUDE.md comments so that future engineers understand why a constraint exists and can evaluate whether it is still appropriate when the context changes: `# Do not read .env files — see Security threat model, Information Disclosure mitigation`.[^3]
- Test CLAUDE.md security constraints periodically: run a session that would violate a documented constraint and verify that the constraint prevents the behavior. Constraints that are not tested may not be enforced as intended, particularly as model versions and CLAUDE.md syntax evolve.

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| AI Threat Surface Awareness | Document AI-specific vulnerability patterns; add to SAST rules and review checklist | Architect |
| STRIDE for AI Sessions | Complete STRIDE analysis for Claude Code; document mitigations per category | Architect |
| Sprint Threat Model Cadence | Add pre-sprint threat model review to sprint planning agenda | Architect |
| CLAUDE.md Integration | Produce CLAUDE.md diff as primary output of each threat model session | Architect |

---

[^1]: METR — "Current AI Safety and Security Practices," February 2026. https://metr.org/blog/2026/02/ai-safety-security-practices
    AI development process as a threat surface; agentic session capability mapping; minimum-footprint principle for permission configuration; threat model triggers for new capability introduction.

[^2]: Anthropic — "Prompt Injection and Security Considerations for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/security
    Prompt injection as a spoofing vector in AI development sessions; untrusted content handling guidance; session scope constraints as the primary mitigation.

[^3]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
    STRIDE applied to agentic code review workflows; privilege escalation in CI-integrated AI sessions; the minimum permission profile argument for agentic development.

[^4]: Sonar — "The AI Code Quality Report," Sonar, 2026. https://www.sonarsource.com/resources/ai-code-quality-report/
    2.74× vulnerability rate for AI-generated code; vulnerability concentration in authentication, input handling, and cryptographic implementation; the training data mechanism for inherited vulnerabilities.

[^5]: Veracode — "Spring 2026 GenAI Code Security Update: Despite Claims, AI Models Are Still Failing Security," March 24, 2026. https://www.veracode.com/blog/spring-2026-genai-code-security/
    45% security test failure rate for AI-generated code; the argument for sprint-level security review of AI-generated components in security-critical paths.

[^6]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    CLAUDE.md as the operational enforcement layer for threat model mitigations; constraint syntax for file access restrictions and code generation prohibitions; security constraint organization patterns.
