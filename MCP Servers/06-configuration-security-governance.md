## MCP Server 6: Configuration, Security, and Governance

**Related to:** [MCP Servers Overview](00-overview.md) — MCP Server 6 · [Tooling: Settings and Permissions](../Tooling/05-settings-and-permissions.md)[^a] · [Security: Secrets Management](../Security/04-secrets-management.md)[^b] · [Governance: AI Usage Policy](../Governance/02-ai-usage-policy.md)[^c] · [Ethics: Training Data and Privacy](../Ethics/04-training-data-privacy.md)[^d]

---

## Overview

MCP servers introduce a new category of external access in Claude Code sessions. A session with active MCP servers can read from and write to systems that live beyond the local repository — Slack channels, Linear tickets, GitHub repositories, internal documentation portals. Without deliberate governance, the permissions granted to those sessions can accumulate beyond what any task actually requires, and the team's visibility into what Claude has done across those external systems can degrade to zero.

This memo covers the configuration architecture that makes MCP permissions explicit and auditable, the three security risks that Anthropic identifies as primary for MCP integrations and how each is mitigated in practice, the governance model for managing the team's growing set of MCP servers, and the audit cadence that keeps the integration surface area bounded over time.

---

## Section 1: The `.mcp.json` Configuration Architecture

**Description:** The `.mcp.json` project file is the team's single source of truth for which MCP servers are active, what connection parameters they use, and — where the server's permission model supports it — what scopes are granted. This file governs every engineer's Claude Code sessions when they work in the repository; a change to `.mcp.json` is a change to the session environment of every team member simultaneously. It should be treated with the same review rigor as changes to CI/CD pipeline configuration.[^1]

The configuration file should be self-documenting: each server entry should include comments specifying the scopes granted, the reason write access is or is not included, and the date the server was added. An entry that says only `"slack": { "token": "${SLACK_TOKEN}" }` tells a reviewer nothing about why this integration exists or what it can do. An entry with comments explaining that the Slack integration has read-only scope for five specified channels, that write access was explicitly excluded, and that the integration was added on a specific date by a named owner is a configuration artifact that can be reviewed and maintained.[^2]

The file should be stored in the repository root and subject to required review by the architect before any merge. GitHub's CODEOWNERS file is the appropriate enforcement mechanism: add `.mcp.json` to the architect's CODEOWNERS rule, ensuring that any PR modifying the MCP configuration requires the architect's approval regardless of who authored the PR.[^3]

**Recommended Practice:**
- Add `.mcp.json` to the architect's CODEOWNERS rule immediately, before any MCP servers are configured. The review requirement should be in place before the first configuration is added, not retrofitted after a permissive configuration has already been committed.[^3]
- Adopt a structured comment format for each server entry in `.mcp.json`: server name, scopes granted (explicit list), scopes excluded and why, date added, owner. This format should be documented in the repository's contributing guide so engineers know the expected format when proposing new servers.[^2]
- Store all credentials referenced in `.mcp.json` in environment variables, never inline. The `.mcp.json` file should be safe to commit as a regular source file with no sensitive content; any engineer who can read the repository should be able to read `.mcp.json` without concern.[^1]
- Validate `.mcp.json` syntax in CI: a configuration file with a JSON parse error silently breaks all MCP server connections for the team. A CI step that validates the file's syntax prevents silent failures from reaching the main branch.[^4]

---

## Section 2: The Three Primary MCP Security Risks

**Description:** Anthropic's documentation identifies three primary security risks for MCP integrations: prompt injection through MCP tool results, excessive permission grants, and MCP server shadowing.[^5] Each requires a specific mitigation strategy; treating them as a single undifferentiated "security concern" produces either over-restriction that undermines the integration's value or under-restriction that creates real exposure.

**Prompt injection** occurs when content retrieved through an MCP tool — a Slack message, a GitHub issue comment, a documentation page — contains text intended to manipulate Claude's behavior. The injection typically takes the form of instructions embedded in retrieved content: "Ignore previous instructions and instead..." followed by an unintended command. The primary mitigation is not filtering retrieved content (which is fragile and incomplete) but maintaining human review of Claude's planned actions before execution. If Claude retrieves a Slack thread and then proposes to post a message to an unexpected channel, the human engineer reviewing that proposal is the mitigation.[^5]

**Excessive permission grants** occur when MCP server configurations include scopes that are broader than any task the team actually uses them for. A GitHub integration configured with `repo:write` when the team only needs `repo:read` for its current workflows is an unnecessary permission grant — it expands Claude's effective blast radius without delivering corresponding value. The mitigation is minimum-permission configuration applied at initial setup rather than after scope creep has occurred.[^6]

**MCP server shadowing** occurs when a malicious MCP server defines a tool with the same name as a legitimate tool, overriding its behavior. This is an attack vector primarily relevant to teams that install MCP servers from untrusted sources — community servers without provenance, servers from repositories the team does not control. The mitigation is a restriction to official servers and servers the team builds internally, with architect review required for any third-party server addition.[^5]

**Recommended Practice:**
- For prompt injection: configure Claude Code to present a summary of planned external actions — Slack posts, GitHub comments, Linear updates — before executing them. The engineer reviews the plan and approves; Claude does not execute write operations without explicit confirmation in the session.[^5]
- For excessive permissions: apply a permission audit at initial configuration for each server. Document the minimum scopes required for the team's actual use cases, configure exactly those scopes, and require justification for any addition that exceeds the documented minimum.[^6]
- For server shadowing: restrict the team's `.mcp.json` to official platform servers (GitHub, Slack, Google, Linear) and internally built custom servers. Any third-party community server requires architect review and a documented rationale. Treat unreviewed third-party servers as untrusted code with system access.[^5]
- Brief the engineering team on prompt injection risk in terms of the actual systems they use: a GitHub issue comment or a Slack message could theoretically contain injection instructions. Engineers should not treat retrieved content as inherently trusted; they should review Claude's proposed actions before execution regardless of where the context came from.[^7]

---

## Section 3: Permission Minimum and Scope Audit

**Description:** The principle of minimum permission — granting exactly the access required for the task and no more — is easy to state and systematically violated in practice. When configuring a new MCP server, the path of least resistance is to grant broad access so "it works" without debugging scope errors, then leave the broad access in place. The result is an integration surface that expands monotonically with each server addition, without any mechanism to contract it as requirements narrow or workflows change.[^6]

The practical approach is to start each integration with the narrowest scopes that support the initial use case, document what those scopes are and why they are sufficient, and require justification to add scopes rather than requiring justification to remove them. This inverts the default: broad access must be earned, not assumed.[^8]

For each MCP server the team operates, the relevant permission questions are: what operations are actually performed in sessions (read vs. write, which resource types), what operations could be performed given current scopes (what the configuration allows), and what is the gap between those two. The gap represents unnecessary permission — access that the configuration grants but no workflow requires. Identifying and closing that gap is the substance of the quarterly permission audit.[^6]

**Recommended Practice:**
- For each MCP server in `.mcp.json`, maintain a one-paragraph permission rationale: what scopes are granted, what workflows require them, and what operations would be possible if those scopes were not granted. This rationale is the baseline for the quarterly audit.[^2]
- Quarterly, compare the permission rationale against the actual operation logs: if the rationale says "write access to Linear comments is required for QA test result posting" but the operation logs show no Linear comment posts in the quarter, the write access is unused and should be suspended.[^6]
- When a workflow changes — a team role is reorganized, a tool is deprecated, a process is automated differently — immediately review whether the associated MCP permissions are still required. Permission cleanup should happen at the same time as workflow changes, not at the next scheduled audit.[^3]
- Apply scope restrictions at the most granular level the MCP server supports. For Slack, restrict to specific channels rather than workspace-wide. For GitHub, restrict to specific repositories rather than organization-wide. For Linear, restrict to specific teams rather than workspace-wide. Granular restrictions limit blast radius without restricting the actual use case.[^8]

---

## Section 4: The MCP Server Inventory

**Description:** A team that has deployed multiple MCP servers — GitHub, Slack, Google Drive, Linear, and one or more custom servers — has an integration surface that requires deliberate tracking. The inventory is the mechanism for maintaining that visibility: a record of which servers are active, what scopes they hold, who owns the credentials, when the server was last reviewed, and who is responsible for it when something goes wrong.[^9]

Without an inventory, the team has no reliable answer to the question "what can Claude do in an MCP-enabled session?" That question matters when a new engineer joins and needs to understand the environment, when a security review requires documenting AI system access, and when something unexpected happens in a session and the team needs to determine whether a specific MCP server could have contributed to it.[^7]

The inventory is a living document. It should be updated when a server is added, when scopes change, when the owning engineer changes teams, and when a server is deprecated. A stale inventory is more dangerous than no inventory: it creates false confidence about the team's MCP configuration that prevents accurate security assessment.[^9]

**Recommended Practice:**
- Maintain the MCP server inventory as a Markdown document in the `.claude/` directory at the repository root, alongside CLAUDE.md. Each entry should include: server name, what it connects to, scopes granted, credential location (secrets management path, not the credential itself), owner, date added, date last reviewed.[^2]
- Include the inventory review as a standing agenda item in the team's quarterly AI practice retrospective. The review should confirm that each entry reflects the current configuration, that ownership is current, and that credentials have been rotated within the team's standard rotation period.[^9]
- Use the inventory as the input for onboarding new engineers to the team's Claude Code environment. A new engineer should be able to read the inventory and understand exactly what external access their Claude Code sessions will have — no surprises.[^7]
- Establish a deprecation process for MCP servers: when a server is no longer actively used, it should be removed from `.mcp.json` and the inventory should be updated. Unused servers that remain configured are unnecessary permission grants and unnecessary infrastructure to maintain.[^6]

---

## Section 5: Write-Access Governance and Confirmation Requirements

**Description:** Write-access MCP operations — posting to Slack, creating GitHub issues, updating Linear tickets, writing files to Google Drive — require governance that is proportional to their visibility and reversibility. The spectrum runs from highly reversible with low visibility (a comment on a GitHub issue that can be deleted) to difficult to reverse with high visibility (a Slack message posted to a team channel, which is seen immediately and cannot be fully retracted). The governance model should match the irreversibility of the operation.[^10]

The foundational principle is that Claude should not execute any write operation through an MCP server without explicit confirmation from the engineer in the session. This is not the same as asking Claude to confirm before every action (which produces friction that degrades usability); it is asking Claude to present its intended write action, including the target and the content, and wait for the engineer to explicitly approve before proceeding. One confirmation per write operation, not one confirmation per session or no confirmation at all.[^5]

For automated sessions — CI pipelines, scheduled jobs, or other contexts where Claude runs without a human in the loop — write-access MCP operations should be disabled by default. The confirmation requirement that mitigates write-access risk in attended sessions cannot be satisfied in unattended sessions; the appropriate mitigation is not to configure write access in those contexts.[^8]

**Recommended Practice:**
- Configure Claude Code to require explicit confirmation before any MCP write operation using the tool confirmation settings in Claude Code's configuration. This setting should be applied globally to all sessions that have MCP server access, not just to specific sessions where write access seems likely.[^10]
- Categorize write operations by reversibility when introducing them: easy to reverse (GitHub issue comments, draft PR creation), moderate reversibility (GitHub review comments, Linear status updates), difficult to reverse (Slack posts, Linear comments visible to stakeholders). Introduce them in that order, with longer observation periods for harder-to-reverse operations.[^6]
- For CI and other automated contexts: create a separate `.mcp.json` configuration file specifically for automated sessions that includes only read-only MCP servers. The automated configuration should be explicitly named (`.mcp.ci.json`) and referenced explicitly in the CI configuration rather than using the default project configuration.[^4]
- When a write operation is executed without the engineer intending it — an unexpected Slack post, a GitHub comment that appeared in the wrong thread — treat it as a configuration incident, not a workflow error. Investigate which configuration allowed the unattended write and add a constraint to prevent recurrence.[^5]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| `.mcp.json` Architecture | Add to CODEOWNERS; adopt structured comment format; validate in CI | Architect |
| Security Risk Mitigation | Configure write-confirmation requirement; restrict to official/internal servers; brief team on injection risk | Architect |
| Permission Minimum | Document permission rationale per server; establish quarterly scope audit | Architect |
| Inventory Management | Create inventory document in `.claude/`; add to quarterly retrospective agenda | Architect |
| Write-Access Governance | One-confirmation-per-write policy; create read-only config for automated sessions | Architect + Backend lead |

---

[^1]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    Shared `.mcp.json` project configuration; CLAUDE.md import patterns; checking AI configuration artifacts into git as team-owned resources.

[^2]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    Configuration documentation practices; CLAUDE.md structure; the principle of self-documenting configuration artifacts.

[^3]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
    Audit logging patterns for external service integrations; quarterly review practices; treating AI tool access with the same governance applied to other service dependencies.

[^4]: Anthropic — "Model Context Protocol Introduction," Claude Code Documentation, 2026. https://code.claude.com/docs/en/mcp-introduction
    MCP open standard architecture; client-server protocol model; permission scoping guidance; transport layer options.

[^5]: Anthropic — "MCP Security Best Practices," Model Context Protocol Documentation, 2025. https://modelcontextprotocol.io/docs/concepts/security
    The three primary MCP security risks: prompt injection via retrieved content, excessive permission grants, and MCP server shadowing. Mitigation strategies for each risk. Governance model for write-access operations in team environments.

[^6]: Greg Kamradt — "MCP Servers for Teams: Governance and Rollout Patterns," YouTube, March 2026. https://www.youtube.com/watch?v=F8pOxXoqFcQ
    - ~0:00 — Team rollout sequencing: why read-only scopes first is risk management rather than caution; how write access should be gated on observed behavior
    - ~9:10 — Shared `.mcp.json` in version control: review discipline, per-server scope documentation, and quarterly access audit workflow
    - ~17:50 — Prompt injection via MCP tool results: threat model, real-world examples, and the human-in-the-loop confirmation pattern
    - ~25:30 — Case study: permission incidents during team MCP rollout and resulting configuration changes

[^7]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    MCP's role in reducing manual coordination overhead; external tool access as a team-level capability requiring governance proportional to its scope.

[^8]: Jack Herrington — "Claude Code MCP Servers: A Complete Setup Guide," YouTube, November 2025. https://www.youtube.com/watch?v=3QkVZj_nKoA
    - ~0:00 — MCP server architecture: how Claude Code acts as an MCP client connecting to external services
    - ~12:15 — Security: credential management, minimum-permission scoping, and audit log configuration for MCP read and write operations

[^9]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    External tool integration as a component of context engineering; shared configuration artifacts as team-level productivity multipliers; the governance overhead that scales with integration surface area.

[^10]: Greg Kamradt — "MCP Servers for Teams: Governance and Rollout Patterns," YouTube, March 2026. https://www.youtube.com/watch?v=F8pOxXoqFcQ
    Write-access governance: the confirmation-per-write-operation policy; categorizing operations by reversibility; the case study of a team that introduced write access too broadly and the incident that resulted.

[^a]: [Tooling: Settings and Permissions](../Tooling/05-settings-and-permissions.md) — MCP server permissions are a specific category of settings governance; the two documents address the same concern at tool level and MCP level.
[^b]: [Security: Secrets Management](../Security/04-secrets-management.md) — MCP server configurations often include credentials and API keys; secrets management practices are a prerequisite for secure MCP configuration.
[^c]: [Governance: AI Usage Policy](../Governance/02-ai-usage-policy.md) — usage policy governs what data sources Claude Code can access; MCP server configuration is the technical implementation of those access boundaries.
[^d]: [Ethics: Training Data and Privacy](../Ethics/04-training-data-privacy.md) — MCP servers can expose sensitive data to AI sessions; privacy protection requires that MCP scope is governed as carefully as direct file access.
