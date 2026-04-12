## MCP Server 2: Slack Integration

**Related to:** [MCP Servers Overview](00-overview.md) — MCP Server 2 · [Tooling: MCP Integration](../Tooling & Configuration/03-mcp-integration.md)[^a] · [Documentation: Knowledge Transfer](../Documentation/03-knowledge-transfer.md)[^b] · [Issues: Prompt Fragmentation](../Issues/07-prompt-fragmentation.md)[^c]

---

## Overview

Slack is the team's primary coordination layer, which means it is also the primary location where engineering context accumulates outside of Claude's view. Incident resolutions, architectural decisions made in threads, product clarifications that happened in a channel discussion, deployment notes posted as follow-ups to PRs — all of this is currently invisible to Claude Code sessions. The Slack MCP server makes that context accessible without requiring engineers to manually locate, copy, and paste thread content into prompts.

This memo covers the specific workflows where Slack context access changes session quality, the permission model for responsible Slack integration, the important distinction between read and write access for a social platform, and the team briefing required before deploying an integration that touches message history.

---

## Section 1: What the Slack MCP Server Enables

**Description:** The official Slack MCP server allows Claude to search message history, retrieve channel conversations, access thread context, look up user information, and — with elevated permissions — post messages and create canvases.[^1] The integration operates through Slack's OAuth system; the team configures a Slack app with specific scopes and shares credentials through the `.mcp.json` project file.

For engineering workflows, the high-value read operations are message search (finding prior discussions on a specific error, decision, or component), thread retrieval (reading the full context of a discussion that produced a decision), and channel history (reading a chronological record of a channel's recent activity). These operations convert Slack from a tool Claude cannot see into a searchable memory the team has already been building.[^2]

The distinction between Slack and other MCP integrations is the social dimension. A GitHub write operation that goes wrong creates an artifact in a repository; it is visible but bounded. A Slack message that goes wrong is posted into a social channel with immediate human visibility, cannot be fully retracted (others may have already read it), and carries the implicit authority of whoever's credentials were used. This asymmetry is why the read/write boundary for Slack deserves more deliberate governance than for purely technical systems.[^3]

**Recommended Practice:**
- Configure the Slack MCP server with read-only OAuth scopes: `channels:history`, `channels:read`, `search:read`, and `users:read`. Explicitly exclude `chat:write` and any posting-capable scopes from the initial configuration.[^1]
- Scope the accessible channels rather than granting workspace-wide read access. Create a curated list of engineering-relevant channels — `#engineering`, `#incidents`, `#architecture`, `#deployments`, `#pr-reviews` — and limit the integration to those. Personal channels and DMs should not be accessible.[^4]
- Store the Slack bot token in the team's secrets management system, not in `.mcp.json` directly. Reference it via environment variable in the configuration file so the credential is not committed to the repository even in an indirect form.[^3]
- Verify the scope of access before rolling out to the team: confirm in the Slack app settings which channels the bot can read and which scopes are active. The configured scopes in `.mcp.json` and the actual Slack app permissions must match.[^1]

---

## Section 2: Debugging with Prior Slack Context

**Description:** The most immediate workflow improvement from the Slack integration is the debugging session pattern: before diagnosing an error, instruct Claude to search relevant Slack channels for prior mentions of that error, component, or symptom. In a team that has been operating for months, many errors have been encountered before. The resolution often lives in an incident thread that was closed, not documented, and subsequently forgotten. The Slack integration converts that forgotten context into recoverable session input.

The concrete mechanism is a session-start search: given an error message, component name, or symptom description, Claude searches the `#incidents`, `#engineering`, and `#deployments` channels for relevant threads, retrieves those threads, and summarizes what prior investigations found and how they were resolved. This step takes under a minute in a session but can collapse hours of re-investigation for errors the team has seen before.

For the QA engineer, the same pattern applies to regression investigation: when a test fails in an unexpected way, searching for prior discussions about that test or the component it covers often surfaces the reason the behavior changed — a deployment note, a configuration change, an intentional behavior modification that was communicated in Slack but not in code comments.[^6]

**Recommended Practice:**
- Establish a session-start convention for debugging: before any diagnosis begins, Claude searches the relevant Slack channels for the error message, component name, and any synonyms or related terms. Document this in CLAUDE.md as a required first step for debugging sessions, not an optional enhancement.[^4]
- Instruct Claude to surface the thread date, the resolution, and whether the resolution was documented elsewhere (in a runbook, an ADR, a JIRA ticket) as part of the search summary. Context that exists only in Slack is context at risk of being lost; the search step is also an audit of documentation gaps.
- When a debugging session finds no relevant Slack context for a significant error, that absence is itself signal: add a step at the close of the debugging session to post a brief summary to the relevant channel so the next engineer who encounters the issue has context. This builds the Slack memory that the integration later retrieves.[^2]
- Configure Claude to distinguish between resolved incidents (which provide resolution context) and open or recurring incidents (which provide diagnostic context). Both are valuable but inform different session strategies.[^6]

---

## Section 3: Decision Context and Architectural History

**Description:** Architectural decisions frequently originate in Slack discussions. A thread in `#architecture` where the team debated two API design approaches, the thread where the database choice was made, the discussion where a library was eliminated from consideration — these decisions are in Slack, not in any document, and they inform implementations that happen months later. Without the Slack integration, Claude treats every implementation as context-free; with it, Claude can read the history of the decisions that constrain the implementation.[^7]

This is particularly high-value for the architect, whose job includes ensuring that Claude's suggestions in sessions are consistent with prior decisions. Rather than needing to relay "we decided against that approach in October" from memory, the architect can instruct Claude to search for the relevant discussion and read the actual thread. The decision's context, the alternatives that were rejected, and the reasoning behind the choice are all preserved in the Slack record.

For the product managers, Slack history provides the conversational record behind product decisions that were made informally. A feature that was scoped down in a product-engineering discussion thread, an acceptance criterion that was modified in a Slack conversation before the spec was updated — these are the constraints that PRDs and Linear tickets sometimes miss, and they are available via the Slack integration.[^8]

**Recommended Practice:**
- When beginning an implementation session that touches an architectural decision point — API design, data model, authentication flow — instruct Claude to search `#architecture` and related channels for prior discussions before proposing implementation approaches. This prevents Claude from re-opening decisions the team has already made.[^7]
- Use the Slack integration to identify decisions that exist only in Slack and have not been formally documented. A session that retrieves a significant architectural decision from a thread is a signal to create an ADR. The integration surfaces documentation debt as a byproduct of normal usage.[^4]
- For product managers running planning sessions, include a Slack search step for the feature being planned: retrieve relevant `#product` and `#product-engineering` threads to surface informal scope decisions, stakeholder feedback, and constraint discussions that may not be reflected in the formal product documentation.[^8]
- Document discovered Slack decisions in the appropriate location — an ADR for architectural choices, a Linear ticket comment for product constraints — so subsequent sessions can retrieve that context from a more durable source rather than relying on the Slack search pattern indefinitely.

---

## Section 4: Write Access and Social Considerations

**Description:** Write access to Slack — the ability for Claude to post messages to channels — is categorically different from read access and should be governed accordingly. A file write is a local operation visible only to the engineer; a Slack message is a social broadcast visible to the team, interpreted as coming from the account whose credentials were used, and not cleanly retractable. The decision to introduce write access is not a technical decision about convenience; it is an organizational decision about trust and social protocol.[^3]

If write access is introduced, the appropriate initial scope is a dedicated bot channel — `#claude-output` or similar — where Claude's posts are understood by all viewers to be AI-generated outputs, not human communications. This channel serves as a staging area where engineers can review Claude's intended post content before it is promoted to a shared team channel. Claude never posts directly to `#engineering`, `#general`, or any channel where its posts would be interpreted as coming from a team member.

The second appropriate write scope is reaction posting — adding emoji reactions to messages rather than posting new messages. This is lower-stakes than text posts, visually distinct from human-authored content, and useful for the QA workflow of marking issue-resolution messages as verified or flagging incident threads as requiring follow-up.[^1]

**Recommended Practice:**
- Do not introduce write access to shared team channels. If write access is needed, create a dedicated output channel and restrict `chat:write` permissions to that channel only using Slack's channel-specific permission model.[^3]
- If Claude posts to Slack, require that the post content be reviewed by the engineer in the session before the post is executed. Claude prepares the draft; the engineer approves and triggers the post. There is no unattended Slack posting in any configuration.
- Use a dedicated Slack app with a clearly named bot identity — "Claude Assistant" or "Engineering Bot" — rather than posting under individual user credentials. Bot-attributed messages are visually distinct in Slack and make it clear to readers that the content was AI-generated.[^1]
- Brief the full team before introducing any Slack write access, including the channel restrictions, the bot identity, and the confirmation-before-post policy. Team members should not be surprised by messages appearing under a bot identity they have not been informed about.[^4]

---

## Section 5: Team Briefing and Privacy Considerations

**Description:** The Slack integration reads message history. Unlike a GitHub or Drive integration that reads technical artifacts, Slack message history includes personal communications, sensitive professional discussions, and content that team members may not have considered would be accessible to an AI system. Before deploying the Slack MCP integration, the team must be explicitly informed of what Claude can access and in what contexts.

The relevant questions for team briefing are: which channels can Claude read in an active session, can Claude read message content posted before the integration was configured (historical access), whether session transcripts that include retrieved Slack content are stored anywhere, and whether individual team members can opt specific messages out of Claude's accessible context. Clear answers to these questions — communicated before the integration is active — allow team members to make informed decisions about what they say in MCP-accessible channels.[^4]

This is not a hypothetical concern. A team member who asks a sensitive question in `#engineering` — about performance feedback, team dynamics, or a confidential initiative — would reasonably expect that message not to be surfaced in a Claude Code session. The integration's channel scope limitation mitigates but does not eliminate this risk. The briefing makes the scope and limitations explicit.

**Recommended Practice:**
- Brief the team in writing before enabling the Slack MCP integration: document which channels are accessible, that read access is limited to those channels, and that Claude does not access DMs or personal conversations. Share this documentation in `#engineering` and in the team's internal wiki.[^4]
- Include the Slack integration scope in the team's AI usage policy document (see [Governance Overview](../Governance/00-overview.md)): which systems Claude can read, what is excluded, and how to request a channel be added or removed from the accessible set.
- Configure the accessible channel list conservatively initially — three to five channels — and expand only after the team has understood and accepted the access model. Adding a channel to Claude's accessible scope is a decision that affects everyone who has ever posted in that channel.[^3]
- Establish a review process for channel scope changes: adding a new channel to the MCP-accessible list requires the channel owner (or the architect) to notify current channel members before the access becomes active.[^4]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Initial Configuration | Configure read-only scopes; limit to 3–5 engineering channels | Architect |
| Debugging Pattern | Add Slack search step to CLAUDE.md debugging convention | Backend lead |
| Decision Context | Establish architecture-search protocol for implementation sessions | Architect |
| Write Access | Create dedicated bot channel if write access introduced; briefing required | Architect |
| Team Briefing | Notify team of accessible channels before enabling the integration | Architect |

---

[^1]: Slack — "Slack MCP Server," Slack Developer Documentation, 2025. https://api.slack.com/docs/mcp
    Official Slack MCP server capabilities: message search, channel history, thread retrieval, write operations. OAuth scope definitions, workspace access controls, and channel permission model.

[^2]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    MCP's role in reducing manual coordination overhead; converting conversational context into accessible session input; context engineering as the primary discipline in agentic development.

[^3]: Anthropic — "MCP Security Best Practices," Model Context Protocol Documentation, 2025. https://modelcontextprotocol.io/docs/concepts/security
    Prompt injection risk, minimum-permission configuration, and the governance model for write-access MCP operations in team environments.

[^4]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    Shared `.mcp.json` project configuration; CLAUDE.md patterns for session conventions; checking AI configuration artifacts into git as team-owned resources.


[^6]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
    Audit logging patterns for external service integrations; quarterly review practices for permission scoping.

[^7]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    External tool integration as context engineering; reducing manual context assembly; shared configuration artifacts as team-level productivity multipliers.

[^8]: Anthropic — "Model Context Protocol Introduction," Claude Code Documentation, 2026. https://code.claude.com/docs/en/mcp-introduction
    MCP open standard architecture; client-server protocol model; permission scoping guidance.


[^a]: [Tooling: MCP Integration](../Tooling & Configuration/03-mcp-integration.md) — MCP integration covers the general configuration discipline; this document applies it to the Slack server specifically.
[^b]: [Documentation: Knowledge Transfer](../Documentation/03-knowledge-transfer.md) — Slack MCP makes conversation-based decisions queryable; it is a tool for capturing the tacit knowledge that knowledge transfer practices depend on.
[^c]: [Issues: Prompt Fragmentation](../Issues/07-prompt-fragmentation.md) — Slack MCP can surface shared prompting patterns from team channels; it is a discovery mechanism for reducing fragmentation.
