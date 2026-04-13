## Settings and Permissions: Governing What Claude Code Can Do

**Related to:** [Tooling Overview](00-overview.md) — Tool 5 · [Governance: AI Usage Policy](../Governance/02-ai-usage-policy.md)[^a] · [Security: Secrets Management](../Security/04-secrets-management.md)[^b] · [MCP Servers: Configuration, Security, and Governance](../MCP%20Servers/06-configuration-security-governance.md)[^c] · [Ethics: Training Data and Privacy](../Ethics/04-training-data-privacy.md)[^d]

---

## Overview

Claude Code's permission system determines which tools Claude can call, which operations it can perform, and under what conditions it must pause and ask for explicit approval. The system is layered — global settings establish defaults, project-level settings narrow or extend them, and session-level flags can narrow them further for specific tasks. For a small team operating with a mix of interactive and unattended sessions, getting this layering right is the difference between a system that engineers trust and one that either interrupts too often or operates with more access than any individual session actually requires.[^1] The minimal-permission principle — granting Claude only the access its current task genuinely needs — is the foundation of both safety and trust.

This memo covers the permission model's composition across layers, how to configure `settings.json` for team use, how to scope permissions specifically for agentic (unattended) sessions, how to define permission profiles for different task types, and how to audit permission scope over time to prevent informal expansion. These practices are not primarily about restricting Claude Code's capabilities — they are about ensuring that the capabilities in use at any moment are intentional, reviewable, and appropriate to the task at hand.[^2]

---

## Section 1: The Permission Model

**Description:** Claude Code's permission model operates through two lists: `allowedTools` and `deniedTools`. Tools on the allowed list can be called by Claude without per-call confirmation; tools on the denied list cannot be called at all in that scope. Tools that appear on neither list require explicit per-call approval — Claude presents the intended action and waits for the engineer to confirm before proceeding. This three-state model (always-allow, always-deny, ask-each-time) gives teams fine-grained control over which operations are routine, which are forbidden, and which require human judgment.[^1]

Permissions compose across three layers: global settings in `~/.claude/settings.json` establish the baseline for all sessions on a machine; project settings in `.claude/settings.json` at the repository root apply to all sessions in that project, overriding globals; and session-level flags (`--allowedTools`, `--deniedTools`) apply to a specific session invocation, overriding both. The composition direction is always narrowing: project settings cannot grant permissions that global settings have denied, and session flags cannot expand permissions beyond what project settings allow. This means the global settings establish the ceiling, and each subsequent layer can only restrict.[^3] The minimal-permission principle follows directly: configure each layer to grant only what that layer's context actually requires.

**Recommended Practice:**
- Document the team's permission model in CLAUDE.md: list which tools are always-allowed, which are always-denied, and which require per-call approval. This makes the permission model visible to everyone who reads the project configuration, not just those who set it up.[^2]
- Apply the minimal-permission principle at every layer: grant a tool always-allow status only if every session in that scope will need it without exception. When in doubt, leave a tool in the ask-each-time state rather than granting always-allow preemptively.[^1]
- Review the denied list at project setup. Tools that should never be available in this repository — direct production database write tools, deployment triggers, credential-modifying operations — belong on the denied list rather than the ask-each-time list. Denied means the tool cannot be called; ask-each-time means it can be called with approval, which is not the same protection level.[^3]
- Use environment-specific settings files (`.claude/settings.staging.json`, `.claude/settings.production.json`) when the same repository is used in multiple deployment contexts with different permission requirements. Load the appropriate file at session start rather than maintaining a single settings file that tries to handle all contexts.

---

## Section 2: Configuring settings.json for Team Use

**Description:** The `settings.json` file at `.claude/settings.json` in the repository root is the primary lever for establishing consistent permissions across the team. When this file is checked into version control, every team member who clones the repository gets the same permission baseline without per-engineer configuration. The file specifies tool permission lists, default model parameters, hook configurations, and MCP server references. Checking it in is the single most important action for ensuring that Claude Code behaves consistently for all team members working in the same codebase.

The key distinction in settings management is between settings that are project properties (belong in version control) and settings that are personal preferences (belong in the user-level `~/.claude/settings.json`). Project properties include tool permission lists, hook configurations, MCP server definitions, and model behavior constraints that reflect the team's engineering standards. Personal preferences include UI layout, preferred response verbosity, personal MCP server additions, and any tool permissions specific to an individual's local setup. Mixing these two categories — putting personal preferences in the project settings file, or keeping project-critical permissions only in personal settings — creates inconsistency that is hard to diagnose when sessions behave differently across engineers.

**Recommended Practice:**
- Check `.claude/settings.json` into version control with the initial project setup. Add a comment at the top of the file explaining that this file is team-standard and personal settings belong in `~/.claude/settings.json`.
- Define the boundary between project settings and personal settings in the team's engineering guide. The rule of thumb: if a setting affects what Claude can do to the codebase, it is a project setting; if it affects how Claude presents information to you, it is a personal setting.
- Treat changes to `.claude/settings.json` with the same review discipline as changes to CI configuration. A PR that modifies the project settings file should explain the reason for the change, the impact on existing sessions, and any related CLAUDE.md updates needed.[^2]
- Add `.claude/settings.json` to the repository's CODEOWNERS file so that proposed changes require review from the Architect or a designated owner before merging. Unreviewed settings changes are one of the more common sources of permission scope expansion.[^1]

---

## Section 3: Agentic Session Permission Scoping

**Description:** Agentic sessions — sessions where Claude Code operates with minimal human supervision, executing multi-step tasks over extended periods — require narrower permission scoping than interactive sessions, not broader. This is counterintuitive to engineers who think of automation as needing more access, but the logic is sound: in an interactive session, an engineer reviews each significant action in real time and can intervene immediately. In an unattended session, an incorrect action can propagate through many steps before anyone notices. A CI-run code review session needs read access to the repository and write access to post PR comments; it does not need write access to the filesystem, deployment tools, or database connections. Granting only those permissions makes the session safer to run unattended.

Broad permissions in agentic sessions are a governance risk because they create a gap between what an authorized human would approve and what the session can do autonomously. If a CI session has filesystem write access and database access because those were convenient to include, a prompt injection vulnerability or a misconfigured skill could trigger operations that no engineer would have approved interactively. The `--allowedTools` flag, passed at session invocation time, is the primary mechanism for narrowing permissions to exactly what a specific agentic workflow requires. For CI integration, this means defining the exact tool set for each pipeline step and passing it explicitly rather than relying on the broader project defaults.

**Recommended Practice:**
- For every agentic session type (CI review, automated security scan, scheduled refactoring), define the minimum tool set the session requires and document it. Start with no tools allowed and add only what the workflow demonstrably needs.
- Pass `--allowedTools` explicitly at every agentic session invocation, even if the project's default settings would permit the same tools. Explicit scoping at invocation time makes the permission intent visible in the CI configuration or script rather than buried in settings files.
- Never grant write access to production systems in agentic sessions unless the specific session type is a deployment workflow with its own approval gates. Code review sessions, security scan sessions, and test-generation sessions should be read-heavy with write access limited to commenting and reporting.[^3]
- Log the `--allowedTools` configuration for every CI session run. Include it in the CI step output so that permission scope is part of the audit trail, not just an implicit property of the run.

---

## Section 4: Permission Profiles for Different Task Types

**Description:** Rather than configuring permissions ad hoc for each new task type, teams benefit from defining a small set of reusable permission profiles that correspond to the most common session categories. Three profiles cover most workflows: a read-only profile for exploration and analysis tasks (file reading, grep, code search — no writes, no execution), a write profile for implementation tasks (file reading and writing, no execution, no external API calls), and an execution profile for test runners and CI tasks (file reading, command execution within defined scopes, no production system access).[^9] Each profile is a named, documented tool list that can be referenced in skill definitions, CI configurations, and team documentation, ensuring that permission decisions are made once at the profile level rather than re-decided for each task.

Profile-based permission management also simplifies onboarding: a new engineer does not need to understand the full permission model to use Claude Code safely — they need to understand which profile applies to which kind of work. "Use the read-only profile when exploring an unfamiliar codebase; use the write profile when implementing a feature; use the execution profile when running the test suite" is a teachable rule that keeps new engineers operating within the team's permission standards without requiring deep familiarity with the underlying configuration.[^10]

**Recommended Practice:**
- Define and document three profiles as the baseline: read-only (exploration), write (implementation), and execution (test/CI). Store the tool lists for each profile as comments in `.claude/settings.json` or as a reference table in CLAUDE.md.[^9]
- When creating new skills or CI workflows, explicitly state which profile they operate under. The skill file should include a line like "This skill operates under the read-only profile" so engineers invoking it know the permission scope.
- Review profile definitions during the quarterly permission audit. As the team's tooling evolves, profiles may need to add or remove tools to stay current. A read-only profile that still includes a tool that now has write side effects is no longer read-only in practice.[^10]
- For high-risk workflows (anything touching production data, external APIs, or deployment systems), create a named profile with explicit documentation of why each tool is included. High-risk profiles should require Architect review before being used in any agentic session.

---

## Section 5: Auditing and Reviewing Permission Changes

**Description:** Permission scope tends to expand informally over time. An engineer adds a tool to the always-allow list because it was inconvenient to approve each time; a CI workflow is updated to include an additional tool without a corresponding review of whether that tool is appropriate for unattended execution; a new skill is added with a broader profile than its task requires. None of these expansions is necessarily malicious, but the cumulative effect is a permission model that no longer reflects intentional design. By the time the scope has expanded significantly, it is difficult to reconstruct which changes were deliberate and which were convenience-driven — and the team has lost confidence in the permission model as a meaningful constraint.

CLAUDE.md serves as a permission governance layer when it explicitly documents the intended permission model alongside the tool configuration. When the documented model and the actual settings diverge, the divergence is visible to anyone reviewing the files. This is most useful during code review: a PR that modifies `.claude/settings.json` without a corresponding update to the permission documentation in CLAUDE.md is a signal that the change may not have been fully considered. The quarterly permission audit exists precisely to close this kind of gap — to compare the intended model with the actual configuration and bring them back into alignment.

**Recommended Practice:**
- Schedule a quarterly permission audit as a standing agenda item. The audit should compare the current `settings.json` tool lists against the documented intent in CLAUDE.md and against the actual task profiles in use. Discrepancies require either a settings change or a documentation update.
- Add a section to CLAUDE.md titled "Permission Model" that lists the always-allowed tools, always-denied tools, and the reasoning for each. Update this section as part of any PR that modifies `settings.json`.[^2]
- Use git history to detect informal permission expansion: at the quarterly audit, review all changes to `.claude/settings.json` since the last audit. Flag any that were not accompanied by a PR description explaining the rationale and any that were merged without Architect review.
- Assign the CTO as the audit owner for the permission model. The quarterly audit is not an engineering task — it is a governance task. Having a non-engineer owner signals that permission scope is a governance concern, not just a configuration convenience.

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| The Permission Model | Document always-allowed, always-denied, and ask-each-time tools in CLAUDE.md; apply minimal-permission principle at every layer | Architect |
| Configuring settings.json for Team Use | Check `.claude/settings.json` into version control; add to CODEOWNERS; document personal vs. project settings boundary | Architect |
| Agentic Session Permission Scoping | Define minimum tool sets for each CI/agentic session type; enforce `--allowedTools` at invocation | Backend lead |
| Permission Profiles for Different Task Types | Define and document read-only, write, and execution profiles; reference profiles in skill definitions and CI configs | Architect |
| Auditing and Reviewing Permission Changes | Schedule quarterly permission audit; add "Permission Model" section to CLAUDE.md; assign audit ownership | CTO |

---

[^1]: Anthropic — "Security and Permissions," Claude Code Documentation, 2026. https://docs.anthropic.com/en/docs/claude-code/security
 Permission model architecture: allowed/denied tool lists, the three-state model (always-allow, always-deny, ask-each-time), composition across global/project/session layers.

[^2]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
 Minimal-permission principle in practice: why granting only the access a task requires is a trust-building practice, not just a security practice; CLAUDE.md as a permission governance layer.

[^3]: Anthropic — "Claude Code Settings Reference," Claude Code Documentation, 2026. https://docs.anthropic.com/en/docs/claude-code/settings
 Settings composition: how global, project, and session-level settings layer and narrow; the `--allowedTools` and `--deniedTools` flags; always-deny vs. ask-each-time protection levels.

[^9]: Dex Horthy (YC Root Access) — "Advanced Context Engineering for Agents," YouTube, August 2025. https://www.youtube.com/watch?v=IS_y40zY-hc
 - Permission profiles for task categories: defining read-only, write, and execution profiles as reusable configurations rather than per-task decisions
 - Profile-based onboarding: how named profiles simplify permission education for new engineers joining a team
 - Profile review during audits: how to detect when a profile's tool list has drifted from its intended scope

[^10]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
 Permission profiles as productivity infrastructure: how named profiles reduce cognitive overhead in daily use; the role of profiles in making permission decisions teachable rather than expert-knowledge-dependent.

[^a]: [Governance: AI Usage Policy](../Governance/02-ai-usage-policy.md) — settings and permissions are the technical enforcement layer for usage policy; policy constraints become permission configurations.

[^b]: [Security: Secrets Management](../Security/04-secrets-management.md) — permission scoping controls what Claude Code can access; secrets management and permission boundaries are the same security concern at different layers.

[^c]: [MCP Servers: Configuration, Security, and Governance](../MCP%20Servers/06-configuration-security-governance.md) — MCP server permissions are a specific category of settings governance; the two documents address the same concern at tool level and MCP level.

[^d]: [Ethics: Training Data and Privacy](../Ethics/04-training-data-privacy.md) — privacy constraints on what data Claude Code can access are implemented through permission settings; policy and technical control are paired here.
