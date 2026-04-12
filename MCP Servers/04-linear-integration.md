## MCP Server 4: Linear Integration

**Related to:** [MCP Servers Overview](00-overview.md) — MCP Server 4 · [Tooling: MCP Integration](../Tooling/03-mcp-integration.md)[^a] · [Governance: Sprint Planning Gates](../Governance/03-sprint-planning-gates.md)[^b] · [Workflows: Task Decomposition](../Workflows/02-task-decomposition.md)[^c]

---

## Overview

Linear is the team's ticketing system, which makes it the single authoritative source for what is being built and why. The context stored in a Linear ticket — requirements, acceptance criteria, comment history, dependencies, status — is precisely the context that should anchor every implementation session. Currently, that context reaches Claude through engineer paraphrasing: the engineer reads the ticket, mentally summarizes it, and describes the task in the prompt. The Linear MCP integration replaces that paraphrase with direct access, eliminating the translation error and the information loss that accumulates in every relay.

This memo covers the implementation and QA workflows where Linear access changes session quality most materially, the session-start convention that converts ticket reading from an optional step to a session default, the write access patterns appropriate for the QA engineer's workflow, and the governance model for Linear bot account management.

---

## Section 1: What the Linear MCP Server Enables

**Description:** The Linear MCP server — available in an official implementation and several well-maintained community versions — exposes issues, comments, project structure, team memberships, labels, milestones, and status transitions to Claude Code sessions.[^1] For the team's existing Linear workflow, the read operations are immediately high-value: Claude can retrieve a ticket's full description, its comment thread, its linked issues, and its acceptance criteria in a single operation at the start of a session.

The architecture of a well-structured Linear ticket contains more implementation signal than an engineer's paraphrase of it. The ticket body has the product requirement; the comments contain the engineering discussion of edge cases, the decision about implementation approach, and the acceptance criteria refinements that accumulated during grooming; the linked issues provide the context for dependencies and prior art. An engineer who has read all of this is better positioned to implement correctly than one who described the ticket from memory. Claude, given direct access, starts from the same complete picture that the fully-prepared engineer does.[^2]

Write operations — status updates, comment creation, issue creation, label assignment — are available and appropriate for specific workflows, particularly the QA engineer's pattern of posting test result summaries to the corresponding issue. These should be introduced selectively and governed through a bot account rather than individual credentials.[^3]

**Recommended Practice:**
- Configure a Linear MCP server with read-only API key access initially: issues, comments, project structure, and team information. Read-only is sufficient for the primary value case — anchoring implementation sessions to ticket context.[^1]
- Verify the integration covers the full ticket structure before rolling out: confirm that Claude retrieves the issue body, all comments (not just the most recent), linked issues, and the current status. A partial retrieval that omits comment history misses the most implementation-critical context.[^2]
- Store the Linear API key in the team's secrets management system; reference it via environment variable in `.mcp.json`. Do not commit API keys, even read-only ones, to the repository.[^4]
- Establish a testing protocol before full rollout: run a session on a known, recently completed ticket and verify that Claude's retrieved context matches what the implementing engineer experienced during the actual implementation. Gaps between the two indicate retrieval limitations to address before relying on the integration.[^3]

---

## Section 2: The Session-Start Convention

**Description:** The primary value of the Linear integration is not the capability itself but the session-start convention that makes it a reliable session default. A capability that engineers must remember to invoke is a capability that will be invoked inconsistently — the engineers who remember to use it will benefit; the others will not. The session-start convention converts ticket reading from a best practice some engineers follow into a required step that all sessions begin with, regardless of which engineer initiates the session.[^5]

The convention is simple: any implementation session that corresponds to a Linear ticket must begin with Claude reading that ticket before any code is written. This instruction belongs in CLAUDE.md as a hard requirement, not a soft suggestion. The session prompt should include the ticket identifier; Claude reads the ticket before responding to any other part of the prompt.

The practical effect is that the implementation Claude produces is anchored to the ticket's actual requirements rather than the engineer's summary of them. For tickets with significant comment history — where the final implementation constraint was established in a comment rather than the original description — this is the difference between an implementation that handles the edge case and one that does not.[^6]

**Recommended Practice:**
- Add the following to CLAUDE.md: "When given a Linear ticket identifier (e.g., ENG-247), read the full ticket — including all comments and linked issues — before beginning any implementation. Do not proceed on the basis of a verbal description of the ticket if the ticket itself is accessible."[^4]
- Train the team to include the Linear ticket identifier in every implementation prompt: "Implement ENG-247" rather than "Implement the authentication timeout feature." The identifier is the link to the authoritative context; the description is a fallback for when the integration is unavailable.[^5]
- When a ticket has linked issues — a parent epic, dependent issues, or blocking issues — instruct Claude to read those as well before beginning. Implementation decisions made without knowledge of a blocking dependency frequently need rework when the dependency constraint is discovered mid-session.[^2]
- For sessions where the ticket has insufficient detail for implementation — vague acceptance criteria, missing edge case handling, unresolved design questions — instruct Claude to identify the gaps before writing any code. Surfacing ambiguity before implementation begins is cheaper than discovering it after the implementation is complete.[^6]

---

## Section 3: QA Workflow Integration

**Description:** The QA engineer's workflow is the highest-leverage application of the Linear integration beyond implementation sessions. Test planning sessions anchored to the ticket's acceptance criteria produce more complete coverage than those anchored to the engineer's memory of the acceptance criteria. The gap between those two starting points is where regression misses originate: a test suite that tests what the engineer remembered from the ticket rather than what the ticket actually specified.[^7]

The specific value for QA is threefold: acceptance criteria are retrieved in full rather than recalled from memory, linked issues are available to reveal edge cases and dependency constraints that affect testing scope, and the comment thread is accessible to surface the implementation decisions that the QA engineer may not have been present for. Together, these inputs produce a test plan that reflects the full scope of what was built and why, not a reconstruction from partial information.

The second QA application is defect reporting: when a test session finds behavior that does not match the ticket's acceptance criteria, Claude can read both the acceptance criteria and the implementation behavior, produce a precise defect description grounded in the ticket, and post that description as a comment on the relevant Linear issue (with write access enabled). This closes the feedback loop between testing and ticketing without requiring manual copy-paste between tools.[^8]

**Recommended Practice:**
- Establish a session-start convention for QA: every test planning session begins with Claude reading the relevant Linear ticket, all comments, and any linked design or acceptance criteria documents from Drive (see [Google Drive Integration](03-google-drive-integration.md)). Document this as a required step in CLAUDE.md.[^4]
- Use the ticket's acceptance criteria as the organizing structure for the test plan: Claude reads the criteria and generates test cases that map directly to each criterion, producing traceability between the test suite and the product requirements without manual effort.[^7]
- For regression testing sessions, instruct Claude to read the tickets for all changes in the release before generating the regression scope. A regression suite generated with direct knowledge of what changed is more precisely targeted than one generated from a general understanding of the release area.[^6]
- When write access is introduced for defect posting, require that Claude prepare the defect description for human review before posting it to the Linear issue. The QA engineer reviews the description, confirms it accurately represents the observed behavior, and triggers the post. There is no unattended defect creation.[^8]

---

## Section 4: Write Access and Bot Account Governance

**Description:** Linear write access — status updates, comment posting, issue creation — is appropriate for the QA engineer's workflow and for certain project management automations, but requires more careful governance than read access. Linear tickets are the team's shared record of what is being built; modifications to tickets by a bot account affect the information that all team members and stakeholders rely on. The governance model must ensure that Claude's write operations are both correct and attributable.[^3]

The appropriate governance model is a dedicated Linear bot account rather than individual engineer credentials. A bot account makes Claude's actions visible in Linear's activity log under a distinct identity — "Engineering Bot" rather than an individual's name — ensuring that Claude-initiated changes are distinguishable from human-initiated changes. If a Claude-posted comment contains an error, it is immediately clear that the comment came from an automated source, not from the engineer whose session produced it.[^9]

Status transitions — marking a ticket as "In Review," "In QA," or "Done" — are a higher-stakes write operation than comment posting. A status transition that is incorrect or premature affects the entire team's view of project progress and may trigger downstream automation. Status transitions by Claude should require explicit engineer confirmation in every case; they should never be configured as an automatic session output.[^3]

**Recommended Practice:**
- Create a dedicated Linear bot account for all Claude-initiated write operations. Configure the bot's permissions at the team level, not the workspace level: it should have access to the engineering team's issues and no others.[^9]
- Introduce write access incrementally: begin with comment posting on issues the engineer has open in session, observe behavior for two weeks, then evaluate whether status transitions are appropriate to enable.[^3]
- For all write operations, require engineer confirmation before execution: Claude proposes the comment or status change, the engineer approves, and the operation executes. There is no configuration where Claude writes to Linear without a human in the loop.[^9]
- Audit the bot account's Linear activity monthly: review which issues received comments, whether the comments were accurate representations of the session's findings, and whether any status transitions were incorrect. Use the audit to identify patterns that warrant additional constraints.[^4]

---

## Section 5: Cross-Tool Context Chaining

**Description:** The Linear integration's full value is realized when it is used in combination with the GitHub and Google Drive integrations. A Linear ticket references a GitHub issue; the GitHub issue has a linked pull request; the pull request closes the ticket. An implementation session that begins by reading the Linear ticket, follows the link to the GitHub issue, reads the comment history there, and then reads the referenced Drive document for architectural context is a session with materially more complete context than one that starts from a verbal description of any individual piece.[^10]

This cross-tool chaining pattern is the practical realization of context engineering: rather than constructing context manually by copying relevant passages from multiple tools into a prompt, the engineer provides the Linear ticket identifier and instructs Claude to follow the relevant links. Claude reads the Linear ticket, identifies linked GitHub issues and Drive documents, retrieves them, and begins implementation from a synthesized understanding of all three.

For the architect reviewing implementation sessions, cross-tool chaining also creates an audit trail: the session's retrieved context is visible in Claude's reasoning, making it possible to verify that the implementation was anchored to the correct ticket, the correct GitHub issue, and the correct architectural documents. This is more verifiable than implementation anchored to an engineer's verbal description.[^6]

**Recommended Practice:**
- When a Linear ticket links to a GitHub issue, instruct Claude to read both before beginning. The ticket has the product framing; the issue has the technical constraints and comment-accumulated edge cases. Together they provide complete context; individually each is partial.[^10]
- When an implementation session requires both Linear context and Drive document context (ADRs, runbooks, API specs), establish the session-start sequence in CLAUDE.md: Linear ticket first, then linked GitHub issue, then relevant Drive documents. This ordering ensures product requirements frame the session before architectural constraints are applied.[^4]
- Use the cross-tool context pattern to generate implementation summaries at session end: given the ticket, the issue, and the implementation produced, Claude writes a brief summary of what was implemented and how it addresses the ticket's requirements. This summary becomes the PR description or the Linear comment that closes the implementation loop.[^6]
- For sessions where the Linear ticket, the GitHub issue, and the Drive documents tell inconsistent stories — different acceptance criteria, conflicting constraints — instruct Claude to surface the inconsistency explicitly before proceeding. Inconsistent context is a pre-implementation signal that requires human resolution.[^5]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Initial Configuration | Configure read-only API key access; test full ticket retrieval | Backend lead |
| Session-Start Convention | Add ticket-read requirement to CLAUDE.md; train team on ticket ID in prompts | Architect |
| QA Workflow | Add QA session-start convention to CLAUDE.md; create test-plan-from-criteria pattern | QA engineer |
| Write Access | Create Linear bot account; introduce comment posting for QA first | Backend lead + QA |
| Cross-Tool Chaining | Document the Linear → GitHub → Drive session-start sequence in CLAUDE.md | Architect |

---

[^1]: Linear — "Linear MCP Server," Linear Developer Documentation, 2025. https://linear.app/docs/mcp
    Linear MCP server capabilities: issue and comment access, project and team structure, status transitions, and comment creation. Authentication via API key and permission model for bot versus user credentials.

[^2]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    MCP's role in reducing manual coordination overhead; how ticket context access changes implementation session quality; context engineering as the primary discipline in agentic development.

[^3]: Greg Kamradt — "MCP Servers for Teams: Governance and Rollout Patterns," YouTube, March 2026. https://www.youtube.com/watch?v=F8pOxXoqFcQ
    - ~0:00 — Team rollout sequencing: read-only first; write access gated on observed behavior
    - ~25:30 — Case study: a small engineering team's Linear MCP rollout, permission incidents, and configuration tightening

[^4]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    Shared `.mcp.json` project configuration; CLAUDE.md session-start convention patterns; checking AI configuration into git as a team-owned resource.

[^5]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    External tool integration as context engineering; reducing manual context assembly; converting optional practices into session defaults through CLAUDE.md configuration.

[^6]: Artur Less — "Spec-Driven Development with Claude Code," Level Up Coding / Medium, March 2026. https://levelup.gitconnected.com/spec-driven-development-with-claude-code-1b08184965e3
    Spec-driven implementation: reading the authoritative specification (the ticket) before writing code as a session discipline; how incomplete context produces implementations that require rework.

[^7]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
    Audit logging for external service integrations; quarterly review of permission scoping; QA workflow integration patterns.

[^8]: Anthropic — "Model Context Protocol Introduction," Claude Code Documentation, 2026. https://code.claude.com/docs/en/mcp-introduction
    MCP architecture; client-server protocol model; write-access governance and human-in-the-loop patterns.

[^9]: Anthropic — "MCP Security Best Practices," Model Context Protocol Documentation, 2025. https://modelcontextprotocol.io/docs/concepts/security
    Minimum-permission configuration; bot account governance for write-access operations; prompt injection risk via retrieved ticket content.

[^10]: Jack Herrington — "Claude Code MCP Servers: A Complete Setup Guide," YouTube, November 2025. https://www.youtube.com/watch?v=3QkVZj_nKoA
    - ~4:30 — Cross-tool MCP configuration: combining Linear, GitHub, and Drive integrations in a single session configuration
    - ~12:15 — Security: credential management for multiple MCP servers; minimum-permission scoping across tool integrations

[^a]: [Tooling: MCP Integration](../Tooling/03-mcp-integration.md) — MCP integration covers the general configuration discipline; this document applies it to the Linear server specifically.
[^b]: [Governance: Sprint Planning Gates](../Governance/03-sprint-planning-gates.md) — Linear MCP provides ticket context to sessions; sprint planning gate classifications can be surfaced within sessions working on classified tickets.
[^c]: [Workflows: Task Decomposition](../Workflows/02-task-decomposition.md) — Linear MCP provides task context that informs decomposition; sessions can query ticket requirements and acceptance criteria as part of task planning.
