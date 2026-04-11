## Overview

For a team of 11 — including 4 backend engineers, 3 frontend engineers, 1 architect, 1 QA engineer, and 2 product managers — the Model Context Protocol represents one of the highest-leverage configuration investments available in the Claude Code ecosystem. MCP is an open standard, introduced by Anthropic in November 2024, that allows Claude Code to communicate with external tools and services through a standardized client-server interface.[^1] Rather than pasting documentation into prompts or switching between tools to gather context, Claude can query live data from the systems the team already depends on — Slack, GitHub, Google Drive, Linear — within the same session.

The practical consequence is a material reduction in the friction that currently sits between intent and implementation. An engineer working a Linear ticket no longer needs to copy ticket context into a prompt; Claude can read the ticket directly. A PR review session no longer requires manually attaching architecture decisions from Google Drive; Claude can retrieve them. A debugging session no longer requires hunting down the relevant Slack thread; Claude can search it. Each of these integrations eliminates a category of manual context assembly that currently absorbs engineering time without producing any value.[^2]

The team already uses Slack, GitHub, Google Drive, and Linear as its primary coordination layer. Each of these platforms either has an official MCP server or a well-maintained community implementation. Additionally, several self-maintained internal sites — documentation portals, internal dashboards, tooling UIs — can be made MCP-accessible with relatively low implementation overhead if the team builds custom servers. The configuration cost for each integration is paid once, in a shared `.mcp.json` checked into git; after that, every engineer's Claude Code sessions inherit the same external context without individual setup.[^3]

Six integration areas are documented below. They progress from read-only connections to the team's core platforms through write-access patterns and finally to custom server development for internal sites. Each section describes the integration, the specific value it delivers given the team's existing toolstack, and a recommended approach to configuration and permission management.

---

## MCP Server 1: GitHub Integration

**Description:** The official GitHub MCP server, maintained by GitHub, provides Claude Code with direct access to repositories, issues, pull requests, branches, and file contents via the GitHub API.[^4] For a team where Claude Code is already involved in writing and reviewing code, closing the loop between the AI session and the repository management layer eliminates one of the most common interruptions in a development workflow: context switching to the GitHub web UI to read an issue, check PR status, or look up a branch.

In practice, this means an engineer can instruct Claude to read a specific issue before implementing a feature — and Claude will pull the issue description, comments, and linked PRs directly, without the engineer assembling that context manually. For PR review sessions, Claude can retrieve the full diff, the PR description, and any existing review comments in a single step. This is especially high-value for the QA engineer and product managers, who spend a disproportionate amount of time in GitHub for review and tracking work that currently requires manual context assembly.[^2]

The GitHub MCP server supports both read and write operations. Write operations — creating issues, opening PRs, adding comments, pushing commits — should be introduced progressively, beginning with read-only access and expanding only after the team has established confidence in Claude's judgment for a given operation type.[^1]

**Proposed Solution:**
- Configure the official GitHub MCP server in the shared `.mcp.json` project file with read-only scopes initially: repository contents, issues, and pull requests. Check this file into git so all team members inherit the configuration without individual setup.[^3]
- Before extending write permissions, define team policy on which write operations are allowed in unattended versus attended sessions. PR creation and comment posting are reasonable attended operations; automated issue modification should require explicit human approval in the session.[^1]
- Use the GitHub integration specifically for issue-driven development sessions: instruct Claude to read the Linear ticket and the corresponding GitHub issue before implementing, ensuring the session begins with complete context rather than an engineer-summarized paraphrase.[^4]
- Review GitHub MCP usage quarterly: log which operations are performed per session, identify any sessions that have modified repository state outside their intended scope, and tighten permission scopes accordingly.[^5]

---

## MCP Server 2: Slack Integration

**Description:** The official Slack MCP server allows Claude Code to read messages, search channel history, retrieve thread context, and — with appropriate permissions — post messages and create canvases.[^6] For a team that uses Slack as its primary coordination layer, this integration addresses a concrete and recurring problem: engineering context is frequently distributed across Slack threads, and that context is currently invisible to Claude Code sessions that need it.

A debugging session benefits from knowing that the same error was reported in #incidents last week and resolved by a specific configuration change. A feature implementation session benefits from reading the relevant product discussion thread in #product rather than receiving a summarized brief. A PR review session benefits from seeing the Slack conversation where the architectural decision being reviewed was originally made. In each case, the value of the Slack integration is converting conversational context — which currently lives outside Claude's view — into accessible session input.[^7]

Write access to Slack carries social implications that file writes do not. A hook that automatically posts to a shared channel is not equivalent to a file write that only the engineer sees. For this reason, Slack write access should be restricted to sessions where the engineer has explicitly initiated and reviewed the post content, and should never be configured as an automated hook output.

**Proposed Solution:**
- Configure the Slack MCP server with read-only access initially: message history, channel search, and thread retrieval. Limit the accessible channels to those with engineering-relevant content rather than granting access to all workspaces.[^6]
- Establish a specific use pattern for Slack search: before beginning any debugging session on a known error, instruct Claude to search relevant Slack channels for prior context on that error. This converts an optional step that engineers often skip under time pressure into a session-default.[^7]
- If write access is introduced, restrict it to a dedicated bot channel rather than granting access to shared team channels. A session that goes wrong should not post to #engineering or #general.[^6]
- Brief the team on the Slack integration explicitly: engineers should understand that Claude can read Slack history in sessions where the integration is active, so they can make informed decisions about what context to invoke.[^5]

---

## MCP Server 3: Google Drive Integration

**Description:** The official Google Drive MCP server provides Claude Code with access to files, documents, spreadsheets, and folders stored in Drive, including the ability to search by content and retrieve document text.[^8] For a team that uses Google Drive as its primary documentation and company resources layer, this integration addresses the gap between the documentation that exists and the documentation that actually informs Claude Code sessions.

Architecture decision records, onboarding documents, runbooks, API specifications, and meeting notes currently live in Drive and are effectively invisible to Claude Code. Engineers who want Claude to honor documented conventions must either paste the relevant content into the prompt or maintain a parallel copy in CLAUDE.md. The Drive MCP server eliminates this duplication: Claude can read the authoritative documentation directly, reducing both the overhead of maintaining CLAUDE.md and the risk of CLAUDE.md diverging from the documents it was meant to summarize.[^3]

For product managers, this integration is particularly high-value: PRDs, feature specs, and decision logs stored in Drive can be referenced directly in planning sessions, giving Claude access to the product context that currently must be communicated through engineer paraphrasing.

**Proposed Solution:**
- Configure the Google Drive MCP server with read-only access scoped to specific shared folders — architecture docs, ADRs, runbooks — rather than granting access to all of Drive. This limits the surface area of sensitive document exposure.[^8]
- Establish a naming and folder convention for Drive documents that are intended to be MCP-accessible: a specific shared folder whose contents Claude Code sessions can reference. This prevents engineers from inadvertently including sensitive personal documents in Claude's accessible context.[^3]
- Use the Drive integration to reduce CLAUDE.md maintenance overhead: instead of copying architectural decisions into CLAUDE.md, add Drive document references that Claude can retrieve when needed. The authoritative document remains the source of truth.[^8]
- For product managers and QA, document the specific Drive documents most useful as session context — PRDs, test plans, acceptance criteria templates — so they can reference them in planning sessions without needing to remember file IDs.[^5]

---

## MCP Server 4: Linear Integration

**Description:** Linear is the team's ticketing system, and the context stored in Linear tickets — requirements, acceptance criteria, comments, status, and linked issues — is exactly the context that should anchor Claude Code implementation sessions. An official Linear MCP server and several well-maintained community implementations are available, providing access to issues, teams, projects, and comments.[^9]

The case for the Linear integration is straightforward: Claude Code should begin every implementation session by reading the ticket it is implementing. Currently, ticket context reaches Claude through engineer paraphrasing — which introduces summarization errors, omits edge cases, and discards the comment history that often contains the most important implementation constraints. The Linear integration replaces paraphrasing with direct access: Claude reads the ticket, understands the full context, and proceeds from the authoritative source.[^2]

For the QA engineer, the Linear integration has additional value: test session context can be anchored to the specific acceptance criteria defined in the ticket rather than relying on verbal summaries. For product managers, it enables planning sessions that cross-reference ticket history and dependencies without manual lookup.[^9]

**Proposed Solution:**
- Configure a Linear MCP server with read-only access covering issues, comments, and project structure. This is sufficient for the primary use case — anchoring implementation sessions to ticket context — without introducing write-access risk.[^9]
- Establish a session-start convention for Linear-tracked work: every implementation session that corresponds to a Linear ticket begins with Claude reading the ticket before any implementation begins. This should be documented in CLAUDE.md as a required step, not an optional one.[^3]
- Write access to Linear — status updates, comment posting, issue creation — can be introduced for the QA engineer's workflow specifically, where automated test result summaries posted to the relevant ticket have clear value and low risk. Scope this to a dedicated Linear bot account rather than an individual engineer's credentials.[^9]
- Review the Linear integration quarterly to identify whether ticket context is meaningfully changing session output quality — measuring the rate at which engineers correct implementation scope errors — and adjust session-start conventions accordingly.[^5]

---

## MCP Server 5: Internal Site Integration (Custom Servers)

**Description:** The team maintains several internal sites — documentation portals, internal dashboards, tooling UIs — that are not served by an existing MCP server implementation. These sites contain operational context, configuration data, and system state that is currently inaccessible to Claude Code sessions. Custom MCP servers, which are standard HTTP servers implementing the MCP protocol, can be built to expose this data in a form Claude Code can consume.[^10]

Building a custom MCP server is a defined engineering task, not an open-ended research project. The MCP specification is public and well-documented; Anthropic provides SDKs for TypeScript and Python that abstract the protocol implementation, leaving only the application logic — defining which resources and tools to expose and how to fetch them — as custom work.[^11] A custom server for an internal documentation site might expose a single search tool and a resource type representing document pages; this can be built in a day and delivers compounding value across every Claude Code session that needs that context.

For the team's backend engineers, building a custom MCP server for a high-value internal site is a straightforward project. For the architect, the decision about which internal sites warrant a custom server should be driven by a concrete question: which sites contain context that engineers currently copy-paste into prompts? Those are the highest-value candidates.[^10]

**Proposed Solution:**
- Audit internal sites for prompt-paste frequency: sites whose content currently appears in Claude Code prompts via manual copy-paste are the highest-value candidates for a custom MCP server. Begin with the single site that is referenced most frequently.[^10]
- Use Anthropic's TypeScript or Python MCP SDK for custom server development. Define a minimal initial server — one or two tools, one resource type — rather than attempting to expose the full site's content in a first version. Incremental expansion is lower-risk than a large initial scope.[^11]
- Host custom MCP servers on the team's internal infrastructure using the same deployment and monitoring patterns as other internal services. Treat the MCP server as a production service: it should have error logging, health checks, and a defined on-call owner.[^10]
- For internal sites with authentication, implement MCP server credential management using the same secrets management approach used for other service credentials — not hardcoded tokens, not developer-local environment variables that cannot be shared.[^5]

---

## MCP Server 6: Configuration, Security, and Governance

**Description:** MCP servers introduce a new category of external access in Claude Code sessions, and that access requires governance that matches the team's existing practices for third-party service integrations. A Claude Code session with an active MCP server can read and potentially write to systems beyond the local repository. Without deliberate configuration, permission scoping, and audit practices, MCP integrations can expand Claude's effective blast radius in ways that are not visible to engineers operating the session.[^12]

The primary configuration artifact is the `.mcp.json` project file, which defines which servers are active, what connection parameters they use, and — where supported — what permission scopes are granted. This file should be checked into git alongside CLAUDE.md and treated with the same review rigor as production configuration.[^3] A poorly scoped MCP configuration is a configuration vulnerability, not just a workflow inefficiency: a Slack integration granted workspace-admin permissions, or a GitHub integration granted write access to all repositories, creates a session risk that is not visible in the code being written.[^12]

Anthropic's guidance on MCP security identifies three primary risk areas: prompt injection through MCP tool results (malicious content in a retrieved document instructing Claude to perform unintended actions), excessive permission grants beyond what the task requires, and MCP server shadowing where a malicious server overrides a legitimate tool definition.[^1]

**Proposed Solution:**
- Treat `.mcp.json` as a production configuration artifact: require architect review for any change to MCP server definitions or permission scopes, the same as changes to CI/CD pipeline configuration.[^3]
- Apply minimum-permission scoping to all MCP server configurations: grant read-only access by default, extend write access only for specific operations with documented justification, and audit whether write access is actually being used in quarterly reviews.[^12]
- Brief the team on prompt injection risk via MCP: engineers should understand that a retrieved Slack message, GitHub issue, or Drive document could theoretically contain instructions intended to manipulate Claude's behavior. Maintaining human review of Claude's planned actions before execution is the primary mitigation.[^1]
- For write-access MCP operations, configure Claude Code to require explicit confirmation before executing any write — do not allow automated write operations via MCP without a human approval step in the session.[^12]
- Maintain an MCP server inventory as part of the team's security documentation: which servers are active, what scopes are granted, who owns the credentials, and when the access was last reviewed. This documentation should live alongside the team's other service dependency records.[^5]

---

## Summary of Recommended Actions

| Integration | Immediate Action | Access Level | Owner |
|---|---|---|---|
| GitHub | Configure official server in `.mcp.json`; read-only scopes first | Read → Write (attended) | Backend lead |
| Slack | Configure official server; restrict to engineering channels | Read-only | Architect |
| Google Drive | Configure official server; scope to architecture/ADR folders | Read-only | Architect |
| Linear | Configure server; establish ticket-read session-start convention | Read → Write (QA bot) | Backend lead + QA |
| Internal Sites | Audit copy-paste frequency; build first custom server | Custom (read-first) | Backend engineer |
| Governance | Add `.mcp.json` to architect review; document server inventory | N/A | Architect |

---

[^1]: Anthropic — "Model Context Protocol Introduction," Claude Code Documentation, 2026. https://code.claude.com/docs/en/mcp-introduction
    MCP open standard architecture, the client-server protocol model, permission scoping guidance, and the three primary MCP security risks: prompt injection, excessive permissions, and server shadowing.

[^2]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    MCP's role in reducing manual coordination overhead; how external tool integration shifts engineering workflows from local-only to service-aware sessions; context engineering as the primary discipline in agentic development.

[^3]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    Shared `.mcp.json` project configuration, CLAUDE.md import patterns for referencing external documentation, and the principle of checking AI configuration artifacts into git as team-owned resources.

[^4]: GitHub — "GitHub MCP Server," GitHub Official Repository, 2025. https://github.com/github/github-mcp-server
    Official GitHub MCP server documentation: supported operations (repositories, issues, pull requests, branches, file contents), authentication setup, and permission scope configuration.

[^5]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
    Audit logging patterns for external service integrations, quarterly review practices for permission scoping, and the argument for treating AI tool access with the same governance applied to other service dependencies.

[^6]: Slack — "Slack MCP Server," Slack Developer Documentation, 2025. https://api.slack.com/docs/mcp
    Official Slack MCP server capabilities: message search, channel history, thread retrieval, and write operations including message posting and canvas creation. Permission scope definitions and workspace access controls.

[^7]: Jack Herrington — "Claude Code MCP Servers: A Complete Setup Guide," YouTube, November 2025. https://www.youtube.com/watch?v=3QkVZj_nKoA
    - ~0:00 — MCP server architecture: how Claude Code acts as an MCP client and connects to external services through the standardized protocol layer
    - ~4:30 — Configuring Slack and GitHub MCP servers: authentication flow, OAuth scoping, and the `.mcp.json` project file structure for team-shared access
    - ~12:15 — Security considerations: credential management, minimum-permission scoping, and audit log configuration for MCP read and write operations
    - ~18:40 — Live demo: building a Postgres MCP server for live schema inspection and parameterized query execution against a development database

[^8]: Google — "Google Drive MCP Server," Google Workspace Developer Documentation, 2025. https://developers.google.com/workspace/mcp
    Official Google Drive MCP server: file and folder access, document content retrieval, search by content, and permission scoping to specific Drive folders. Authentication via OAuth and service account configuration.

[^9]: Linear — "Linear MCP Server," Linear Developer Documentation, 2025. https://linear.app/docs/mcp
    Linear MCP server capabilities: issue and comment access, project and team structure, status transitions, and comment creation. Authentication via API key and permission model for bot versus user credentials.

[^10]: Anthropic — "Building MCP Servers," Model Context Protocol Documentation, 2025. https://modelcontextprotocol.io/docs/concepts/servers
    Custom MCP server architecture: resource types, tool definitions, prompt templates, and the progression from minimal viable server to full-featured integration. Guidance on scoping initial server implementations.

[^11]: Fireship — "Model Context Protocol Explained in 100 Seconds," YouTube, January 2025. https://www.youtube.com/watch?v=7j_NE6Pjv-E
    - ~0:00 — MCP protocol overview: the client-server model, JSON-RPC transport layer, and how Claude Code acts as an MCP client connecting to external service endpoints
    - ~0:45 — The three MCP primitives — resources, tools, and prompts — and how each maps to practical integration patterns
    - ~1:20 — Building a minimal custom server in TypeScript using the official Anthropic SDK: the minimal viable server scaffold in under 50 lines

[^12]: Anthropic — "MCP Security Best Practices," Model Context Protocol Documentation, 2025. https://modelcontextprotocol.io/docs/concepts/security
    Prompt injection risk via retrieved content, minimum-permission configuration guidance, MCP server shadowing and how to detect it, and the governance model for write-access MCP operations in team environments.

[^13]: David Ondrej — "Build Your Own MCP Server from Scratch (Full Tutorial)," YouTube, February 2026. https://www.youtube.com/watch?v=y4CaCldMDXs
    - ~0:00 — Project setup: initializing the TypeScript MCP SDK and defining the initial server skeleton with resource and tool handler boilerplate
    - ~8:20 — Connecting a custom internal documentation site: search tool implementation, document resource type definition, and authentication against internal APIs using bearer tokens
    - ~22:45 — Deploying to internal infrastructure: health check endpoints, structured error logging, and team credential management without hardcoded secrets
    - ~31:00 — Testing the integration end-to-end: verifying Claude Code can discover and invoke the custom server's tools within a live session

[^14]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    External tool integration as a component of context engineering; the argument that reducing manual context assembly is as important as prompt quality; shared configuration artifacts as team-level productivity multipliers.

[^15]: Greg Kamradt — "MCP Servers for Teams: Governance and Rollout Patterns," YouTube, March 2026. https://www.youtube.com/watch?v=F8pOxXoqFcQ
    - ~0:00 — Team rollout sequencing for MCP integrations: why starting with read-only scopes is risk management rather than caution, and how write access should be gated on observed behavior
    - ~9:10 — Shared `.mcp.json` in version control: review discipline, per-server scope documentation, and the quarterly access audit workflow that keeps permissions bounded
    - ~17:50 — Prompt injection via MCP tool results: the threat model, real-world examples of malicious content in retrieved documents, and the human-in-the-loop confirmation pattern that mitigates it
    - ~25:30 — Case study: a small engineering team's rollout of GitHub and Linear MCP servers, including the permission incidents they encountered and how they tightened configuration
