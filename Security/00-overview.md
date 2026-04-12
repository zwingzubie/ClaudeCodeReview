## Overview

Security guidance for AI-assisted development is currently scattered across five sections of this repository — Ethics documents the accountability structure for AI-generated vulnerabilities, Governance documents the policy requirements for security review, Issues documents comprehension debt and the risks of unreviewed AI output, Tooling & Configuration documents Claude Code hooks and permission management, and Metrics documents vulnerability trends as a health signal. That distribution made sense as each section was built, but it creates a navigation problem: an engineer responding to a security incident, a QA engineer setting up scanning, or an architect designing a security review process has to synthesize guidance from five separate locations. This section consolidates that guidance into a single coherent reference.[^1]

The security case for a unified treatment is not organizational tidiness. AI-assisted development changes the team's security posture in ways that require coordinated response across threat modeling, tooling, dependency management, secrets handling, vulnerability response, and review process. Each of these areas is individually addressable; the security benefit comes from addressing them as a system. A team that runs SAST but does not model the threat surface of AI sessions, that has good secrets hygiene but no vulnerability response procedure, or that has a review checklist but no DAST integration is partially protected — and partial protection in security often means that the unaddressed area becomes the attack vector.[^2]

Sonar's 2026 research found that AI now writes 42% of code in repositories that have adopted AI tools, and that AI-generated code contains security vulnerabilities at 2.74 times the rate of human-written code.[^3] For a team of 11 producing AI-assisted output at that penetration rate, the expected volume of AI-introduced security issues is not marginal — it is a primary security concern that warrants the same deliberate engineering the team applies to functional correctness. The six areas documented below are the structural response to that concern.

---

## Security Area 1: Threat Modeling for AI-Assisted Development

**Description:** Traditional threat modeling assumes that the threat surface is defined by the application's attack vectors — its inputs, its integrations, and its authentication boundaries. AI-assisted development adds a new threat surface: the development process itself. Claude Code sessions process repository content, can execute commands, can write to arbitrary files, and can be manipulated through prompt injection in files the session reads. These are not theoretical risks — they are the operational profile of a tool that has significant capability and reads broadly from the codebase. Failing to model them leaves a class of threats entirely unaddressed.

**Proposed Solution:**
- Conduct a one-time threat modeling session for Claude Code itself: what can it read, what can it write, what commands can it execute, and under what conditions? Map these capabilities against the team's sensitive assets — production credentials, customer data, proprietary business logic — and document the mitigations in place.
- Apply STRIDE to AI development sessions: model spoofing (prompt injection in repository files), tampering (AI-generated code that modifies security-critical paths unexpectedly), information disclosure (sessions that read more than intended), and privilege escalation (sessions run with permissions broader than the task requires).[^5]
- Establish a threat modeling cadence for AI-heavy sprints: the architect reviews the threat implications of significant new AI-generated components before they merge to main, not after they deploy to production.
- Encode threat model outcomes as CLAUDE.md constraints: if the threat model identifies that AI should not read certain directories, generate certain patterns, or modify certain files, those constraints belong in CLAUDE.md where they are enforced at the session level rather than depended on in a document no one reads during execution.[^6]

---

## Security Area 2: SAST/DAST Integration

**Description:** AI-generated code failing security tests at 2.74 times the rate of human-written code is not an argument against AI adoption — it is an argument for calibrating security tooling to the actual risk profile of AI output. Teams that adopted AI tools without adjusting their scanning configurations are running security tooling designed for a lower-vulnerability code mix against a higher-vulnerability actual mix. The result is a false sense of security: the pipeline still passes, but the pass threshold is calibrated to a risk level the team no longer inhabits.[^3]

**Proposed Solution:**
- Deploy SAST tooling (Semgrep, CodeQL, or Snyk Code) in the CI pipeline with blocking thresholds for critical and high findings. This is the baseline — SAST on every PR, blocking on high severity, with no bypass for AI-generated PRs.[^7]
- Configure Claude Code Stop event hooks to run a targeted SAST scan before any AI-generated code reaches the human review stage. Pre-commit SAST catches the class of issues that human reviewers are least likely to notice in AI-generated code: injection sinks, insecure deserialization patterns, and hardcoded secrets.[^8]
- Add DAST to the testing pipeline for API endpoints generated by AI. SAST finds static patterns; DAST finds runtime vulnerabilities that only surface under actual request conditions. AI-generated REST and GraphQL endpoints are particularly prone to authorization logic errors that SAST cannot detect but DAST can surface.
- Review SAST rule sets quarterly and update configurations to include AI-specific vulnerability patterns as they are identified by the security research community.

---

## Security Area 3: Dependency and Supply Chain Security

**Description:** AI coding tools suggest dependencies based on training data that has a cutoff date and does not reflect the current vulnerability landscape. A dependency that was uncontroversial in training data may have accumulated critical CVEs since. AI also tends to suggest packages by convenience — how commonly they appear in similar code — rather than by security posture. On a small team without a dedicated security engineer, there is no natural checkpoint where suggested dependencies are evaluated before they enter the lockfile.[^10]

**Proposed Solution:**
- Run automated dependency scanning (Dependabot, Snyk, or npm audit / pip-audit / bundler-audit) as a CI pipeline step with blocking thresholds for critical findings. Dependency vulnerabilities that are not caught in CI are not caught until a breach or an external audit.[^11]
- Add a CLAUDE.md constraint prohibiting AI from adding new dependencies without explicit human approval: "Do not add new packages to package.json, requirements.txt, go.mod, or equivalent lockfiles without explicit instruction. Propose new dependencies and wait for approval." This prevents the silent accumulation of AI-suggested dependencies.[^6]
- Maintain lockfile discipline: commit lockfiles to version control, review lockfile diffs as part of PR review, and treat unexplained lockfile changes as a signal requiring investigation rather than a routine update to approve without reading.[^11]
- Before accepting any AI-suggested dependency, evaluate it against four criteria: published maintainer, recent activity, known CVE history, and license compatibility. This is a two-minute check for any individual dependency that prevents the category of supply chain compromise where an AI suggests a plausible-looking but malicious or abandoned package.[^10]

---

## Security Area 4: Secrets Management

**Description:** Claude Code sessions read files to understand context. In a repository that does not rigorously separate secrets from code — or in an engineer's local environment where credentials exist alongside the working directory — there is a non-trivial risk that session context includes credentials, API keys, or other secrets. The risk is not limited to accidental generation of code containing secrets; it includes the possibility that session logs, shared context, or model outputs reflect sensitive values that were present in the session's input context.[^12]

**Proposed Solution:**
- Maintain a `.claudeignore` file that explicitly excludes files and directories containing secrets from Claude Code's read access: `.env`, `.env.*`, `secrets/`, credential files, and any directory used for local certificate storage. This file is the session-level analog to `.gitignore` — it prevents the information from entering the session rather than depending on the session to not reproduce it.[^8]
- Deploy secrets scanning as a pre-commit hook (using tools such as `detect-secrets`, `trufflehog`, or `git-secrets`) and as a CI pipeline step. Pre-commit scanning catches secrets before they reach the remote repository; CI scanning catches cases where the pre-commit hook was bypassed or misconfigured.[^13]
- Enforce environment variable patterns for all credentials: no credentials in source files, no credentials in configuration files that are committed to version control, no credentials in test fixtures. Document this as a CLAUDE.md constraint so that AI-generated code defaults to environment variable references rather than inline values.[^6]
- Conduct a secrets audit of the repository at least annually: use `trufflehog` or `git-secrets` against the full commit history to identify any credentials that were committed and not subsequently rotated. AI-generated code is not the only path through which secrets reach repositories, but it is a path that warrants systematic checking.

---

## Security Area 5: Vulnerability Response Procedures

**Description:** A team that discovers a security vulnerability in AI-generated code faces two simultaneous challenges: responding to the vulnerability itself and understanding how it bypassed the review process. The second challenge is unique to AI-assisted development — the retrospective question is not just "what was wrong with this code" but "what does AI-generated code in this domain tend to get wrong, and how should CLAUDE.md be updated to prevent recurrence?" Without a defined response procedure, both questions tend to be answered poorly under the time pressure of incident response.[^14]

**Proposed Solution:**
- Define four severity levels with response timelines: Critical (authentication bypass, RCE, data exposure) — 4-hour response, immediate escalation to architect and CTO; High (privilege escalation, significant data leakage) — 24-hour response; Medium (non-critical information disclosure, configuration weakness) — 1-week response; Low (defense-in-depth improvement, hardening opportunity) — next sprint planning.[^15]
- Include an AI-origin analysis step in every vulnerability post-mortem: was this generated by AI? If so, what was the prompting context, what pattern did the AI use, and what CLAUDE.md constraint would have prevented it? This step converts individual incidents into systemic improvements.
- Maintain a team vulnerability log that records all security findings, their origin (AI-generated vs. human-written), their severity, and how they were resolved. This log is the data source for the security health metrics tracked in the Metrics section.[^16]
- Define a disclosure and communication procedure that does not require drafting under time pressure: internal discovery to engineering response (same-day for Critical/High), engineering response to customer notification (per contractual SLAs), and customer notification to public disclosure (per applicable regulation). The procedure should be documented before it is needed.

---

## Security Area 6: Security Review Checklist

**Description:** Ad-hoc security review produces inconsistent results — not because reviewers are careless, but because security review requires systematic attention to a category list that is too long to reliably reconstruct from memory under the time pressure of PR review. A standardized checklist ensures that every review covers the same categories and that no category is skipped because the reviewer was focused on a different issue. For AI-generated code, where the vulnerability patterns are less predictable than for human-written code, the systematic coverage a checklist provides is especially valuable.[^2]

**Proposed Solution:**
- Define a security review checklist organized by category: input validation and sanitization, authentication and session management, authorization and access control, cryptographic operations, injection sinks (SQL, command, path traversal), dependency additions, and secrets and credential handling. Each category should have 3–5 specific checks.[^17]
- Implement the checklist as a Claude Code slash command (`/security-review`) that injects the checklist as a structured review prompt. This makes the checklist available at the point of use without requiring engineers to navigate to a separate document.[^8]
- Integrate checklist results into the PR process via a CI step that comments a checklist summary on each PR containing AI-generated code in security-critical paths. The comment serves as both a reminder and a documentation artifact.
- Review and update the checklist quarterly to incorporate newly identified AI-specific vulnerability patterns, findings from the team's own vulnerability log, and guidance from the security research community.

---

## Summary of Recommended Actions

| Security Area | Immediate Action | Owner |
|---|---|---|
| Threat Modeling | Conduct Claude Code threat modeling session; encode outcomes in CLAUDE.md | Architect |
| SAST/DAST Integration | Deploy Semgrep or CodeQL in CI with blocking thresholds; configure Stop event hook | Backend lead |
| Dependency Security | Enable Dependabot or Snyk; add CLAUDE.md dependency constraint | Architect |
| Secrets Management | Create `.claudeignore`; add pre-commit secrets scanning hook | Backend lead |
| Vulnerability Response | Define severity levels and response timelines; create vulnerability log | Architect + CTO |
| Security Review Checklist | Draft checklist; implement `/security-review` slash command | Architect |

---

[^1]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    CLAUDE.md as the central mechanism for encoding security constraints; session configuration as the operational enforcement layer for security policy.

[^2]: Gartner — "Predicts 2026: Software Engineering and DevSecOps," Gartner Research, January 2026. https://www.gartner.com/en/documents/predicts-2026-software-engineering-devsecops
    The security governance gap in AI-assisted development; why partial security coverage under AI adoption creates higher risk than pre-AI baselines; the case for systematic rather than ad-hoc security treatment.

[^3]: Sonar — "The AI Code Quality Report," Sonar, 2026. https://www.sonarsource.com/resources/ai-code-quality-report/
    AI writes 42% of code in AI-adopting repositories; AI-generated code has a 2.74× higher vulnerability rate than human-written code; the calibration argument for adjusted security tooling.


[^5]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
    STRIDE applied to AI development sessions; prompt injection as a spoofing vector; the architecture of a CI pattern that accounts for AI-specific threat categories.

[^6]: Anthropic — "Claude Code: Settings and Configuration," Claude Code Documentation, 2026. https://code.claude.com/docs/en/settings
    CLAUDE.md constraint syntax; `.claudeignore` configuration; Stop event hook configuration for pre-commit scanning integration.

[^7]: Veracode — "Spring 2026 GenAI Code Security Update: Despite Claims, AI Models Are Still Failing Security," March 24, 2026. https://www.veracode.com/blog/spring-2026-genai-code-security/
    45% failure rate on security tests for AI-generated code; the argument for SAST with blocking thresholds calibrated to AI output risk profiles.

[^8]: Anthropic — "Claude Code Hooks Reference," Claude Code Documentation, 2026. https://code.claude.com/docs/en/hooks
    Stop event hook configuration for pre-commit SAST; slash command implementation; `.claudeignore` file format and precedence rules.


[^10]: GitHub — "Octoverse 2025: The State of AI in Software Development," GitHub Octoverse 2025. https://github.blog/news-insights/research/the-state-of-open-source-and-ai/
    AI dependency suggestion patterns; supply chain security risks in AI-suggested packages; the training data cutoff problem for dependency security evaluation.

[^11]: Snyk — "The State of Open Source Security 2025," Snyk, 2025. https://snyk.io/reports/open-source-security/
    Automated dependency scanning effectiveness; lockfile discipline as a supply chain security practice; CVE remediation timelines for open-source dependencies.

[^12]: Anthropic — "Privacy and Data Handling," Claude Code Documentation, 2026. https://code.claude.com/docs/en/privacy-data-handling
    Session context and secrets exposure risk; data retention policies for session inputs; guidance on credential handling in AI-assisted workflows.

[^13]: GitGuardian — "State of Secrets Sprawl 2025," GitGuardian, 2025. https://www.gitguardian.com/state-of-secrets-sprawl
    Secrets in version control: detection rates, remediation rates, and the time-to-exploitation distribution for publicly exposed credentials; pre-commit scanning as the highest-leverage intervention.

[^14]: Dark Reading — "AI-Assisted Development: The Security Risks Nobody Is Managing," October 2025. https://www.darkreading.com/application-security/ai-assisted-development-security-risks
    Vulnerability response challenges unique to AI-generated code; the retrospective analysis gap; velocity pressure as a contributing factor in AI-related security incidents.

[^15]: NIST — "Cybersecurity Framework 2.0," NIST, 2024. https://www.nist.gov/cyberframework
    Severity classification frameworks for vulnerability response; response timeline standards; the Respond function of the NIST CSF as the organizing structure for team vulnerability procedures.

[^16]: CodeRabbit — "State of AI Code Generation: AI vs. Human Code Report," December 17, 2025. https://www.coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report
    Security finding rate comparison between AI-generated and human-written code; vulnerability logging as a data source for security health metrics and trend analysis.

[^17]: OWASP — "OWASP Top 10 2021," OWASP Foundation. https://owasp.org/Top10/
    Security review category taxonomy: input validation, authentication, authorization, cryptography, injection, and the OWASP Top 10 as the canonical checklist foundation for application security review.
