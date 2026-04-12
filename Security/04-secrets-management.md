## Secrets Management in AI-Assisted Workflows

**Related to:** [Security Overview](00-overview.md) — Security Area 4

---

## Overview

Secrets management has always been a source of engineering security failures — exposed API keys, committed credentials, and hardcoded passwords are among the most common sources of security incidents in software development. AI-assisted development adds a new dimension to this risk: the AI development session itself is a processing context that reads files from the working directory, and in environments without explicit access controls, that context may include credentials, API keys, and other sensitive values. Claude Code does not store session data persistently, but the session window during which credentials are present in context is a new exposure surface that did not exist before AI adoption.[^1]

GitGuardian's 2025 State of Secrets Sprawl report found that secrets committed to public repositories are exploited within an average of 24 hours of exposure. The detection-to-exploitation window is not getting longer as AI tools make it easier for adversaries to scan repositories at scale. For a team where AI-generated code may reference environment variables, configuration files, or other credential sources, the risk of an AI session inadvertently producing output that includes a credential — or of a developer inadvertently including credentials in the context they provide to a session — warrants the same systematic treatment as secrets hygiene in the codebase itself.[^2]

The response to this risk has three layers: preventing secrets from entering AI session context in the first place (`.claudeignore` and session scope discipline), preventing secrets from being committed to version control (pre-commit hooks and CI scanning), and ensuring that AI-generated code uses environment variable patterns rather than hardcoded credential references. These layers are complementary, not redundant — a failure at any one layer is still caught by the others.

---

## Section 1: The Specific Risk of Claude Code Reading Secrets

**Description:** Claude Code's context window includes the content of files it reads during a session. In a repository with good secrets hygiene — no credentials in committed files, `.env` files excluded from version control — this presents limited risk. In real engineering environments, however, secrets hygiene is rarely perfect. Engineers work with `.env` files in their local working directories, test fixtures may contain realistic-looking credentials, CI configuration files may reference secrets in ways that Claude Code reads as literal values, and history may contain credentials that were committed and later removed but remain in git history readable by tools that traverse the repository.[^3]

The risk is not primarily that Claude Code will exfiltrate secrets — Anthropic's data handling policy does not use session inputs for training, and the session data handling is documented. The risk is that a session producing a code suggestion, a test file, or a documentation snippet may incorporate credential values it read from context into its output, and that output may then be committed to version control in a form that was not reviewed as carefully as the original source file. An engineer reviewing a generated test file may not notice that a fixture value in the generated output matches an actual API key from a `.env` file the session read earlier.[^1]

**Recommended Practice:**
- Create and maintain a `.claudeignore` file at the repository root that explicitly excludes all credential-containing files and directories: `.env`, `.env.*`, `.env.local`, `.env.production`, `secrets/`, `credentials/`, `*.pem`, `*.key`, `*.p12`, `config/secrets.yml` (or equivalent for the team's stack). This file follows the same syntax as `.gitignore` and prevents the listed paths from entering Claude Code's session context.[^4]
- Add `.claudeignore` to the team's repository template and to the engineer onboarding checklist. A `.claudeignore` file that does not exist in an engineer's local environment provides no protection; it must be present in every environment where Claude Code runs.[^4]
- Apply session scope discipline: when starting a Claude Code session for a task that does not require credentials, start the session from a subdirectory that does not contain credential files rather than from the repository root. Narrower scope is a defense-in-depth measure against credential exposure even when `.claudeignore` is configured correctly.
- Review `.claudeignore` whenever the team's secrets storage patterns change: if a new type of credential file is introduced (e.g., a new cloud provider's credential format), add it to `.claudeignore` before it appears in any engineer's working directory.

---

## Section 2: .gitignore and .claudeignore Discipline

**Description:** `.gitignore` prevents files from being committed to version control; `.claudeignore` prevents files from being read by Claude Code. These are complementary protections: `.gitignore` addresses the version control exposure path and `.claudeignore` addresses the AI session exposure path. A file that is correctly `.gitignore`d but not `.claudeignore`d is still visible to Claude Code sessions. A file that is correctly `.claudeignore`d but not `.gitignore`d is still at risk of accidental commit. Both files must be maintained consistently to close the secrets exposure paths the team controls.[^3]

For most teams, `.gitignore` configurations are already present and reasonably maintained. The gap is typically `.claudeignore` — it is a newer file type that requires deliberate adoption and may not be included in existing repository templates or toolchain configurations. The team's initial `.claudeignore` can be bootstrapped from its `.gitignore`: any path that is `.gitignore`d because it contains secrets should also be in `.claudeignore`. The inverse is not necessarily true — some files that should be excluded from AI session context (large binary files, generated files, vendor directories) are not secrets but are still appropriate `.claudeignore` entries for performance reasons.

**Recommended Practice:**
- Review the team's `.gitignore` files and create corresponding `.claudeignore` entries for all paths excluded because they contain credentials, API keys, or other sensitive values. This is a one-time synchronization step that establishes the baseline; subsequent additions to `.gitignore` for security reasons should be mirrored to `.claudeignore`.[^4]
- Add a CI check that verifies `.claudeignore` contains the entries in a team-maintained required-exclusions list. This check ensures that the `.claudeignore` is not accidentally simplified or overwritten without removing required entries.[^3]
- For engineers using multiple repositories or working across several workspaces, document the personal `~/.claude/` global settings as a place to add exclusion patterns that should apply across all repositories — particularly useful for patterns like `*.pem` and `*.key` that represent credential files regardless of repository.[^4]
- Include `.claudeignore` in the repository alongside `.gitignore` — it is a team configuration artifact that belongs in version control, not a personal configuration that each engineer maintains independently. Committing it ensures consistency across the team.

---

## Section 3: Secrets Scanning Hooks

**Description:** Pre-commit hooks that run secrets scanning before any file is committed to version control are one of the highest-leverage security controls a small team can deploy: they catch the most common secrets exposure path (accidental commit) at the earliest possible intervention point (before the commit exists in the repository). The tools available for this purpose — `detect-secrets`, `trufflehog`, and `git-secrets` — each maintain pattern libraries of common credential formats (AWS access keys, GitHub tokens, Stripe API keys, private keys, database connection strings) and block commits that match those patterns.[^2]

CI-level secrets scanning provides a second layer of protection: it catches cases where the pre-commit hook was not installed, was bypassed, or missed a pattern that the CI tool's database includes. GitGuardian's managed service offers real-time repository scanning with notification on detection, which is appropriate for teams that need faster detection than their scan cadence provides. For a team of 11 without a dedicated security engineer, the pre-commit hook plus CI scan combination provides reasonable coverage without requiring a managed service.

**Recommended Practice:**
- Install `detect-secrets` or `trufflehog` as a pre-commit hook using the `pre-commit` framework. Include the hook configuration in the repository's `.pre-commit-config.yaml` so that running `pre-commit install` during developer onboarding installs it automatically.[^5]
- Configure the CI pipeline to run `trufflehog filesystem --only-verified .` or equivalent as a scan step on every PR, using the `--only-verified` flag to reduce false positives on high-entropy strings that are not actual credentials. The CI scan should run on the full diff, not just the modified files, to catch secrets in newly introduced binary or configuration files.[^2]
- Establish a secrets rotation procedure for the event that a secret is found committed to the repository: immediate rotation of the affected credential (do not wait until the secret is removed from history), removal from history using `git filter-repo`, and notification to any services that used the credential. Document this procedure before it is needed so that the response is not improvised under incident pressure.[^3]
- Run a one-time historical scan of the team's repositories using `trufflehog git` to identify any credentials that may have been committed in the past and not yet rotated. Historical scans are a point-in-time check; establish a quarterly cadence for recurring historical scans as the baseline.

---

## Section 4: Environment Variable Patterns vs. Hardcoded Credentials

**Description:** AI-generated code has a default tendency to use concrete values in examples and placeholders in patterns that look syntactically complete. When generating code that requires credentials — database connection strings, API client initialization, authentication configuration — AI may generate code with placeholder strings (`"your-api-key-here"`) that are close enough to real credentials that an engineer pastes in an actual value without recognizing that the proper pattern is an environment variable reference rather than a literal value. This is not a deliberate AI behavior; it reflects the training data distribution, where code examples frequently use string literals for clarity.[^6]

The environment variable pattern — reading credentials from `os.environ`, `process.env`, or the platform-equivalent at runtime rather than embedding them in source code — is the correct pattern for all credential handling in production-deployed code. It is also not the pattern AI reliably generates by default. CLAUDE.md constraints that specify this pattern explicitly ensure that AI-generated code uses environment variable references from the start rather than generating literal credential patterns that must then be identified and corrected.

**Recommended Practice:**
- Add an explicit CLAUDE.md constraint for credential handling: `When generating code that requires credentials, API keys, database connection strings, or other sensitive values, always use environment variable references (os.environ.get(), process.env, etc.) rather than string literals. Never use placeholder credential strings in generated code.`[^4]
- Create a team standard for environment variable naming and document it in CLAUDE.md: `API keys follow the pattern {SERVICE}_API_KEY; database URLs follow the pattern {SERVICE}_DATABASE_URL; secrets follow the pattern {SERVICE}_SECRET.` A consistent naming convention reduces the chance of environment variable references being incorrect in AI-generated code.[^6]
- Include credential pattern checking in the security review checklist (see Security Overview — Area 6): every PR review for AI-generated code involving external services should verify that credential references use environment variables and that no placeholder strings were left in place.[^3]
- For local development environments, use a credential management tool (1Password CLI, HashiCorp Vault, AWS Secrets Manager with local profile) rather than `.env` files wherever feasible. When `.env` files are necessary, ensure they are in `.gitignore`, `.claudeignore`, and documented as not to be committed under any circumstances.

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| .claudeignore Configuration | Create and commit `.claudeignore` with all credential file patterns | Backend lead |
| .gitignore Synchronization | Audit `.gitignore` for security-motivated entries; mirror to `.claudeignore` | Backend lead |
| Pre-Commit Secrets Scanning | Install `detect-secrets` via `.pre-commit-config.yaml`; add to onboarding | Backend lead |
| Environment Variable Pattern | Add CLAUDE.md credential constraint; include in security review checklist | Architect |

---

[^1]: Anthropic — "Privacy and Data Handling," Claude Code Documentation, 2026. https://code.claude.com/docs/en/privacy-data-handling
    Session context and credential exposure risk; Anthropic data retention policies; the specific risk of credentials appearing in AI-generated output when present in session context.

[^2]: GitGuardian — "State of Secrets Sprawl 2025," GitGuardian, 2025. https://www.gitguardian.com/state-of-secrets-sprawl
    24-hour exploitation window for publicly exposed secrets; detection-to-exploitation statistics; pre-commit hook effectiveness data; CI-level scanning as a second-layer control.

[^3]: Gartner — "Predicts 2026: Software Engineering and DevSecOps," Gartner Research, January 2026. https://www.gartner.com/en/documents/predicts-2026-software-engineering-devsecops
    Secrets hygiene as a baseline security requirement; the CI pipeline integration pattern for secrets scanning; the `.gitignore` and `.claudeignore` synchronization requirement.

[^4]: Anthropic — "Claude Code: Settings and Configuration," Claude Code Documentation, 2026. https://code.claude.com/docs/en/settings
    `.claudeignore` file format and syntax; global settings for cross-repository exclusion patterns; CLAUDE.md credential constraint syntax.

[^5]: pre-commit — "pre-commit: A Framework for Managing and Maintaining Multi-Language Pre-Commit Hooks," 2026. https://pre-commit.com/
    Pre-commit framework configuration; `.pre-commit-config.yaml` syntax; `detect-secrets` and `trufflehog` hook integration; developer onboarding automation via `pre-commit install`.

[^6]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    CLAUDE.md constraint syntax for credential handling patterns; the mechanism by which session-level constraints override AI default code generation behavior for security-critical patterns.
