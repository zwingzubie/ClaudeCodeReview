## Dependency and Supply Chain Security

**Related to:** [Security Overview](00-overview.md) — Security Area 3 · [Issues: Security Vulnerabilities](../Issues/04-security-vulnerabilities.md)[^a] · [Tooling: CI/CD Integration](../Tooling & Configuration/06-cicd-integration.md)[^b] · [Governance: Compliance and Audit](../Governance/06-compliance-and-audit.md)[^c] · [Ethics: Intellectual Property](../Ethics/01-intellectual-property.md)[^d]

---

## Overview

Dependencies are one of the most consequential security decisions an engineering team makes — and AI-assisted development has introduced a new mechanism through which insecure or malicious dependencies enter codebases: AI suggestion. Claude Code and other AI coding tools routinely suggest packages to solve implementation problems. Those suggestions are drawn from training data with a cutoff date, weighted by frequency of appearance in similar code, and presented with the same confidence as syntactically correct generated code. There is no signal in the suggestion itself that distinguishes a well-maintained, actively patched package from one that has been abandoned, compromised, or accumulated CVEs since the training cutoff.[^1]

The supply chain security problem predates AI adoption — the SolarWinds compromise in 2020, the Log4Shell vulnerability in 2021, and the XZ Utils backdoor in 2024 established that the dependency chain is an adversarial surface that sophisticated attackers actively target. AI adoption does not change the nature of supply chain risk; it changes the frequency and mechanism through which new dependencies enter the codebase. On a small team where package management decisions are made quickly under velocity pressure, AI suggestions that bypass a deliberate dependency review process are a meaningful incremental risk.[^2]

The response to this risk is not to refuse AI dependency suggestions — AI suggestions are often reasonable starting points. The response is to ensure that AI suggestions go through the same evaluation process as any other new dependency: maintainer credibility, recent activity, CVE history, and license compatibility. The additional step is making that evaluation process fast enough that engineers do not skip it under time pressure, and making it a CLAUDE.md constraint so that AI cannot add dependencies without the evaluation occurring.

---

## Section 1: AI Tendency to Suggest Outdated or Vulnerable Dependencies

**Description:** AI models suggest dependencies based on their training data, which has a fixed cutoff. A package that was the recommended solution for a problem in 2023 or 2024 may have accumulated critical CVEs, been abandoned by its maintainers, or been superseded by a more secure alternative by the time an engineer uses it in 2026. The AI has no mechanism to know this. It suggests packages that were well-regarded at training time with no visibility into the security landscape that has developed since. This is not a model limitation that will be resolved by model improvements — it is an architectural property of how training works, and it will remain a property of any model with a training cutoff.[^3]

There is a second, more acute risk: typosquatting and dependency confusion attacks. These attacks create malicious packages with names similar to legitimate ones (typosquatting) or with names that match internal package names in public registries (dependency confusion). AI models trained on code that includes such packages, or on text that mentions malicious package names as legitimate alternatives, can suggest malicious packages to engineers who are not aware of the attack vector. Multiple security researchers have documented AI models suggesting non-existent package names that could be registered by an attacker — a variant called "AI hallucination-based dependency confusion."[^1]

**Recommended Practice:**
- Before installing any AI-suggested package, verify it exists in the expected registry, matches the name exactly (check for transpositions), and has a maintainer profile consistent with a legitimate project. A package without a public maintainer, without recent activity, and without a credible repository is a dependency confusion or typosquatting risk regardless of how it was suggested.[^1]
- Run `npm audit`, `pip-audit`, `bundler-audit`, or the equivalent for the team's package ecosystems on every AI-suggested dependency addition before committing the lockfile. The audit will not catch zero-day vulnerabilities, but it will catch CVEs that are already in the advisory database — which covers the vast majority of known-bad AI-suggested packages.[^4]
- Document in the team's engineering handbook that AI-suggested dependencies are not pre-vetted and require the same evaluation as any other new package. This framing should be explicit: the fact that Claude Code suggested it does not constitute an endorsement of the package's security or maintenance status.
- Track AI-suggested packages that failed the dependency review process in the team's vulnerability log. This data reveals whether AI is systematically suggesting problematic packages from a particular ecosystem or category — a pattern that warrants a targeted CLAUDE.md constraint.

---

## Section 2: Automated Dependency Scanning

**Description:** Manual dependency review is necessary but not sufficient. The CVE landscape changes continuously: a package that was clean at last review may have accumulated vulnerabilities since. Automated dependency scanning runs the current codebase's dependency manifest against an up-to-date vulnerability database on a defined cadence, producing alerts when dependencies with known vulnerabilities are detected. For a small team, this scanning is the difference between discovering a critical dependency vulnerability before deployment and discovering it via a breach notification.[^4]

GitHub Dependabot (free for GitHub-hosted repositories), Snyk (free tier covers most small-team needs), and registry-native audit tools (`npm audit`, `pip-audit`, `bundler-audit`) each provide automated scanning at different levels of integration depth. Dependabot is the lowest-friction starting point for teams already on GitHub: it runs automatically, opens PRs for dependency updates, and requires no additional configuration for basic vulnerability scanning. Snyk provides richer context — exploitability scores, remediation guidance, license compliance — but requires a subscription for team-scale features. Registry-native tools provide the most current vulnerability data for their specific ecosystem and are appropriate for CI pipeline integration.

**Recommended Practice:**
- Enable GitHub Dependabot for all repositories as an immediate baseline action. Configure it to run daily rather than weekly for repositories where dependencies are frequently updated or where the team's vulnerability response SLA requires faster detection.[^5]
- Add `npm audit --audit-level=high` (or equivalent) as a CI pipeline step that blocks PR merge on high and critical findings. This step catches vulnerability-carrying dependency updates that Dependabot's PR-based flow might not flag before they are merged by another path.[^4]
- Configure Dependabot or Snyk alerts to route to the engineer who most recently modified the dependency in question, not to a shared channel where they are likely to be ignored. Ownership-routed alerts have significantly higher resolution rates than shared-channel alerts.[^5]
- Set a maximum dependency age policy for security-critical dependencies: core authentication libraries, cryptographic packages, and HTTP client libraries should be updated within one sprint of a fix becoming available for a high or critical CVE. Document this policy so that Dependabot PRs for these packages are treated with appropriate urgency.

---

## Section 3: Lockfile Discipline and CLAUDE.md Constraints

**Description:** A lockfile records the exact resolved versions of all direct and transitive dependencies. When lockfiles are committed to version control and reviewed as part of PR review, the team has visibility into every dependency change: which packages were added, which were updated, and which were removed. When lockfiles are not committed, or when lockfile diffs are approved without review, the team loses visibility into transitive dependency changes — which are how most supply chain compromises occur. An attacker who compromises a transitive dependency (a dependency of a dependency) does not appear in the direct dependency manifest; only the lockfile captures the change.[^2]

Claude Code's default behavior is to generate code that references packages by name without necessarily managing the lockfile. In an interactive session, an engineer may accept a dependency addition without noticing that the lockfile has also changed in ways that include new transitive dependencies. CLAUDE.md constraints that require explicit approval for any lockfile change prevent this silent accumulation and ensure that transitive dependency changes receive the same visibility as direct dependency changes.

**Recommended Practice:**
- Commit lockfiles to version control for all package ecosystems: `package-lock.json` or `yarn.lock` for Node.js, `Pipfile.lock` or `poetry.lock` for Python, `go.sum` for Go. Configure the CI pipeline to fail if the lockfile is not consistent with the dependency manifest — this catches cases where a developer updated dependencies without updating the lockfile.[^2]
- Add an explicit CLAUDE.md constraint prohibiting dependency additions without engineer approval: `Do not add, remove, or upgrade packages in package.json, requirements.txt, go.mod, or any equivalent manifest without explicit instruction. If a package is needed, propose the addition and wait for approval before modifying any manifest or lockfile.` This constraint prevents Claude Code from silently accumulating dependencies during a session.[^6]
- During PR review, treat lockfile diffs as a security artifact: review them for unexpected new packages, unexpected version changes to security-critical dependencies, and unexpected transitive dependency additions. A PR that adds one direct dependency but changes fifteen transitive dependencies warrants investigation before merge.[^5]
- Run `npm ci` (or equivalent) rather than `npm install` in CI environments. The `ci` variant installs exactly what is in the lockfile rather than resolving fresh, ensuring that CI tests the same dependency set the engineer tested locally rather than a potentially different set produced by fresh resolution.

---

## Section 4: Reviewing AI-Suggested Dependencies Before Accepting

**Description:** The review process for AI-suggested dependencies does not need to be time-consuming to be effective. The goal is not a comprehensive security audit of every suggested package — it is a two-minute check that catches the most likely failure modes: abandoned packages, packages with recent critical CVEs, and packages with names similar to well-known packages (typosquatting). Most AI-suggested dependencies will pass this check quickly. The value of the process is in the minority that do not — and in creating a documented record that the team took a deliberate decision rather than installing whatever the AI suggested.[^1]

The four-criteria framework (maintainer credibility, recent activity, CVE history, license compatibility) provides a systematic basis for this review. Maintainer credibility: does the package have a public maintainer profile, a repository with genuine commit history, and community recognition (downloads, issues, stars)? Recent activity: has the package received commits or releases in the last 12 months? A package that was last updated in 2022 is unlikely to be receiving security patches. CVE history: does the package appear in any current CVE database entry? License compatibility: is the package's license compatible with the team's distribution model and customer contracts?

**Recommended Practice:**
- Use `npx package-audit-tool` or equivalent, or simply check the package's page on the relevant registry, for each AI-suggested dependency before installation. The review takes two minutes for a legitimate, well-maintained package and will take longer for a package that warrants investigation — which is the correct signal.[^4]
- Maintain a team-approved dependency list in the engineering handbook for common categories (HTTP clients, logging frameworks, testing utilities, ORM libraries). AI-suggested packages that match a team-approved alternative should be replaced with the approved version rather than adding a new package for the same function.[^3]
- When an AI-suggested package is rejected after review, document the rejection reason and the alternative used in the PR. This documentation builds the team's institutional knowledge about which package categories AI suggestions are reliable in and which require extra scrutiny.[^1]
- Brief the team on AI hallucination-based dependency confusion: AI models can suggest package names that do not exist but look plausible. Always verify that a suggested package exists in the registry before treating it as a valid suggestion — a package that does not exist is an invitation to register a malicious version under that name.

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| AI Suggestion Awareness | Brief team on AI dependency risk; add to onboarding documentation | Architect |
| Automated Scanning | Enable Dependabot; add `npm audit` to CI pipeline | Backend lead |
| Lockfile Discipline | Commit all lockfiles; add CLAUDE.md dependency constraint | Backend lead |
| Pre-Accept Review | Define four-criteria review checklist; create team-approved dependency list | Architect |

---

[^1]: GitHub — "Octoverse 2025: The State of AI in Software Development," GitHub Octoverse 2025. https://github.blog/news-insights/research/the-state-of-open-source-and-ai/
    AI-suggested dependency risks; typosquatting and dependency confusion vectors; AI hallucination-based dependency confusion; training data cutoff as a structural source of outdated suggestions.

[^2]: Snyk — "The State of Open Source Security 2025," Snyk, 2025. https://snyk.io/reports/open-source-security/
    Supply chain attack surface; transitive dependency risk; lockfile discipline as a visibility mechanism; the SolarWinds and Log4Shell precedents for dependency chain compromise.

[^3]: Sonar — "The AI Code Quality Report," Sonar, 2026. https://www.sonarsource.com/resources/ai-code-quality-report/
    AI training data cutoff as a mechanism for outdated dependency suggestions; the frequency-based selection mechanism that favors popular but potentially unmaintained packages.

[^4]: OWASP — "OWASP Dependency-Check," OWASP Foundation. https://owasp.org/www-project-dependency-check/
    Automated dependency scanning tooling; CVE database integration; CI pipeline integration patterns for dependency audit; severity thresholds for blocking PR merge.

[^5]: GitHub — "Dependabot Documentation," GitHub Docs, 2026. https://docs.github.com/en/code-security/dependabot
    Dependabot configuration for daily scanning cadence; ownership-routed alert configuration; Dependabot PR behavior and merge policies for security updates.

[^6]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    CLAUDE.md dependency constraint syntax; the operational mechanism for preventing silent dependency additions during AI development sessions.

[^a]: [Issues: Security Vulnerabilities](../Issues/04-security-vulnerabilities.md) — dependency vulnerabilities are a primary category of the supply chain risk described there; AI-generated dependency selections are a specific threat vector.

[^b]: [Tooling: CI/CD Integration](../Tooling & Configuration/06-cicd-integration.md) — dependency scanning runs in the CI/CD pipeline as an automated gate; the integration document describes the pipeline where dependency checks execute.

[^c]: [Governance: Compliance and Audit](../Governance/06-compliance-and-audit.md) — dependency license compliance is a recurring audit category; license risk in AI-generated dependency selections is a compliance documentation requirement.

[^d]: [Ethics: Intellectual Property](../Ethics/01-intellectual-property.md) — dependency license selection is a primary source of IP risk; the ethical analysis of license compliance applies directly to AI-generated dependency choices.
