## MCP Server 5: Internal Site Integration (Custom Servers)

**Related to:** [MCP Servers Overview](00-overview.md) — MCP Server 5 · [Tooling: MCP Integration](../Tooling/03-mcp-integration.md)[^a] · [Security: Secrets Management](../Security/04-secrets-management.md)[^b] · [MCP Servers: Configuration, Security, and Governance](06-configuration-security-governance.md)[^c]

---

## Overview

The four platform integrations — GitHub, Slack, Google Drive, Linear — address the team's external coordination layer. The internal site integration addresses a different category: operational context that lives on systems the team owns and maintains, and that is therefore not served by any off-the-shelf MCP server. Documentation portals, internal dashboards, configuration UIs, API explorers, internal service registries — these sites contain context that engineers currently copy-paste into prompts manually, and they are exactly the sites where a custom MCP server delivers the most compounding value.

Building a custom MCP server is a defined engineering task, not a research project. The MCP specification is public and well-documented; Anthropic provides SDKs in TypeScript and Python that reduce the implementation to application logic — deciding what to expose and how to fetch it. This memo covers the prioritization method for identifying which internal sites warrant a custom server, the implementation approach using the official SDKs, the deployment model for treating custom servers as production services, and the governance requirements for credential and access management.

---

## Section 1: Identifying High-Value Candidates

**Description:** Not every internal site warrants a custom MCP server. The implementation cost — one to two days for a minimal server, a week for a more capable one — should be applied where the recurring value is highest. The selection criterion is straightforward: which internal sites currently appear in Claude Code prompts via manual copy-paste? Those sites are already providing value in sessions; the question is whether engineers are paying an unnecessary manual tax to provide it.[^1]

The prioritization audit is simple: for one week, have engineers flag any instance where they copy content from an internal site into a Claude Code prompt. At the end of the week, the site that appears most frequently is the highest-priority custom server candidate. This is a concrete, falsifiable measure that avoids the speculative "this might be useful" reasoning that produces tools nobody uses.

For most engineering teams, the top candidates fall into one of three categories: internal documentation portals (technical reference docs that are not in Drive or the repository), internal API explorers or schema browsers (service documentation that is maintained programmatically and would require constant manual updates to keep current in CLAUDE.md), and internal dashboards or monitoring tools where engineers need to describe system state to Claude rather than having Claude see it directly.[^2]

**Recommended Practice:**
- Run the one-week copy-paste audit before building anything. Have engineers add a comment to their session logs when they manually copy internal site content into a prompt. Review the audit at the end of the week and build the server for the highest-frequency site.[^1]
- Prioritize sites whose content changes frequently. A static internal wiki that is updated quarterly provides less value as a custom server than a dynamic service registry that reflects the current state of deployed services. The integration's value scales with content freshness.[^3]
- Explicitly exclude sites where the implementation cost would exceed the value: a site visited once per quarter by one engineer does not warrant a custom server, even if the manual copy-paste is inconvenient. The question is compounding value across all sessions, not convenience for a single workflow.[^2]
- Involve the backend engineers who maintain the candidate sites in the prioritization decision. They understand the site's data model, the API access patterns, and the maintenance overhead that a custom MCP server would introduce. Their input prevents the selection of a technically complex candidate as the first server.[^4]

---

## Section 2: Building a Minimal First Server

**Description:** The correct scope for a first custom MCP server is minimal: one or two tools, one resource type, and coverage of the single most common use case rather than the full site's content. A minimal server that works reliably delivers value immediately and can be expanded incrementally. A ambitious first server that requires three weeks to build, has edge cases the team did not anticipate, and requires debugging before it can be used is a liability, not an asset.[^5]

The MCP protocol exposes three primitive types: resources (data objects Claude can read, analogous to files), tools (functions Claude can invoke with parameters), and prompts (templates that structure how Claude interacts with the server). For a documentation site, the minimal implementation is one search tool and one resource type representing document pages. For a service registry, it is one query tool that returns service metadata for a given service name. For a monitoring dashboard, it is one tool that returns current system state for a specified component.[^6]

Anthropic's TypeScript and Python SDKs handle the protocol implementation — transport layer, JSON-RPC message handling, capability negotiation — leaving only the application logic as custom work. A backend engineer familiar with the target site's API can implement the minimal server in a day using the SDK scaffold, spend a second day connecting it to the site's authentication and data retrieval, and deliver a working integration by the end of the first week.[^7]

**Recommended Practice:**
- Use the official Anthropic MCP SDK (TypeScript or Python, matching the team's preferred backend language) as the implementation foundation. The SDK provides the server scaffold; the engineer fills in the tools and resource handlers. Do not implement the MCP protocol from scratch.[^7]
- Define the server's scope in a one-page spec before writing any code: what tools are exposed, what parameters they accept, what data they return, and what the authentication model is. Review this spec with the architect before implementation begins. Scope changes after implementation are expensive; scope decisions before implementation are cheap.[^4]
- Implement one tool first and test it end-to-end in a Claude Code session before adding additional tools. The most common source of wasted effort in custom server development is building multiple tools, discovering a foundational issue with the transport or authentication layer, and having to rebuild them all.[^5]
- Define success criteria before building: what specific session workflow should the server enable, and how will the team verify it works? A server that retrieves documentation pages successfully is a meaningful milestone; a server that builds without errors is not.[^1]

---

## Section 3: Deployment and Operations

**Description:** A custom MCP server that engineers depend on in daily sessions is a production service. It should be treated as one: deployed on internal infrastructure, monitored with health checks, logged with structured error output, and assigned an on-call owner who responds when it fails. A server that is deployed as a local script on one engineer's machine, or as an undocumented process on an unmonitored host, will fail silently and degrade session quality in ways that are difficult to diagnose.[^8]

The transport layer for team-shared MCP servers is HTTP with server-sent events (SSE) for the production deployment — not the stdio transport used for local development. The SSE transport allows the server to run as a persistent service that multiple engineers' Claude Code sessions can connect to, rather than requiring each engineer to run the server locally.[^6]

Deployment infrastructure for a custom MCP server should match the team's existing internal service deployment patterns: the same container registry, the same orchestration platform, the same secrets management approach used for other internal services. Treating the MCP server as a special case that runs outside normal infrastructure creates operational debt that compounds as the team adds more custom servers.[^9]

**Recommended Practice:**
- Deploy custom MCP servers using the team's existing container deployment infrastructure. Create a Dockerfile during development rather than as a post-implementation step; a containerized server is deployable to the team's existing infrastructure without further configuration work.[^8]
- Implement health check endpoints (`/health` or `/status`) that return the server's operational status and connectivity to its upstream data source. Configure the team's monitoring system to alert the on-call owner when the health check fails.[^9]
- Use structured logging (JSON-formatted log entries with consistent fields: timestamp, operation, parameters, duration, error if any) so the server's activity is queryable through the team's existing log infrastructure. Unstructured logs are unmonitorable at scale.[^8]
- Document the server in the team's internal service registry — the same location where other internal services are documented. Include: what the server exposes, what authentication it requires, which team owns it, and how to report issues. This documentation is the prerequisite for other engineers relying on the server.[^4]

---

## Section 4: Authentication and Credential Management

**Description:** Internal sites have authentication requirements, and the custom MCP server must satisfy them to retrieve data. The credential management approach for a custom MCP server follows the same principles as credential management for any internal service: credentials are stored in the team's secrets management system, not hardcoded in the server code, not in environment variables that live only on one engineer's machine, and not committed to the repository in any form.[^10]

For sites that authenticate via API key, the server reads the key from the secrets management system at startup and uses it for all requests. For sites that require OAuth, the server implements the OAuth client flow using credentials stored in secrets management, with token refresh handled automatically. For sites with mTLS or certificate-based authentication, the server manages the certificate material through the same secrets infrastructure used for other TLS certificates.[^9]

The specific risk for MCP servers is that credential compromise does not just affect the engineer whose machine was compromised — it affects all engineers whose Claude Code sessions connect to the server. A shared server with a compromised credential is a more significant security event than a compromised individual credential. This elevated blast radius justifies more careful credential hygiene, not less.[^10]

**Recommended Practice:**
- During development, use environment variables for credentials. During deployment, migrate to the team's secrets management system (Vault, AWS Secrets Manager, or equivalent). The transition should happen before the server is shared with the full team — not after.[^4]
- Do not commit any credential material to the repository, including example credentials, test credentials, or commented-out credentials. Use `.gitignore` to exclude any credential files and validate this in CI.[^10]
- Implement credential rotation support in the server: when credentials are rotated in the secrets management system, the server should pick up the new credentials without requiring a restart. Hardcoded credential loading at startup creates rotation downtime that the team will eventually observe in session failures.[^9]
- Audit server credential access quarterly alongside the broader MCP server inventory review (see [Configuration, Security, and Governance](06-configuration-security-governance.md)): verify which credentials are in use, whether they have been rotated within the team's standard rotation period, and whether access to the upstream site is still scoped as intended.[^8]

---

## Section 5: Incremental Expansion and Maintenance

**Description:** A custom MCP server that is deployed and working is not a project that is complete. It is the beginning of an investment that compounds as the team uses it, discovers its limits, and expands its capabilities. The maintenance model should anticipate incremental expansion: new tools added as new use cases are identified, performance improvements as the team discovers that certain queries are slow, and deprecation of tools that were built for workflows the team has abandoned.[^5]

The expansion process mirrors the initial build: identify the highest-frequency manual copy-paste that the server does not yet handle, scope a minimal addition (one tool, one parameter type), implement and test it, deploy it. This is a lower-cost process than the initial build because the server infrastructure is already in place; the engineer is only adding application logic, not rebuilding the transport and authentication layers.[^7]

For maintenance, the server's error logs are the primary signal: errors that repeat across multiple sessions indicate either a data source problem (the upstream site changed its API or returned unexpected data) or a tool design problem (engineers are using the tool in ways its parameter definitions do not handle cleanly). Regular review of error logs — monthly, or after any change to the upstream site — prevents silent degradation from accumulating.[^8]

**Recommended Practice:**
- Maintain a backlog of candidate additions for each custom server, ordered by the same copy-paste frequency criterion used for the initial build. Review this backlog monthly and add one tool per sprint when a team member has capacity.[^1]
- When the upstream site changes its API — a version upgrade, a schema change, an authentication change — treat the required server update as a P1 maintenance task. A server that fails silently is worse than no server at all; it produces misleading context in sessions without making the problem visible.[^5]
- Assign each custom server a named owner — the backend engineer who built it or the one who knows the upstream system best. Ownership should be documented in the service registry and in a `OWNERS` file in the server's repository. Ownership without documentation is ownership that evaporates when the engineer changes teams.[^4]
- Set a deprecation threshold: if a custom server's tool is not invoked in any session during a 90-day period, evaluate whether the tool should be deprecated. Tools that are not used are maintenance overhead without value; removing them reduces the server's operational surface area.[^9]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Candidate Identification | Run one-week copy-paste audit; identify top candidate site | Architect |
| Minimal First Server | Define one-page spec; implement one tool first; test end-to-end | Backend engineer |
| Deployment | Containerize during development; deploy to existing infrastructure | Backend engineer |
| Credential Management | Use secrets management from the start; no hardcoded credentials | Backend engineer |
| Maintenance | Assign named owner; add to monthly MCP server review agenda | Architect |

---

[^1]: Anthropic — "Building MCP Servers," Model Context Protocol Documentation, 2025. https://modelcontextprotocol.io/docs/concepts/servers
    Custom MCP server architecture: resource types, tool definitions, prompt templates, and the progression from minimal viable server to full-featured integration. Guidance on scoping initial server implementations.

[^2]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    Context engineering as the primary discipline; how external tool integration shifts workflows; shared configuration artifacts as team-level productivity multipliers.

[^3]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    External tool integration as context engineering; reducing manual context assembly; the argument that reducing friction is as important as prompt quality.

[^4]: Greg Kamradt — "MCP Servers for Teams: Governance and Rollout Patterns," YouTube, March 2026. https://www.youtube.com/watch?v=F8pOxXoqFcQ
    - ~0:00 — Team rollout sequencing and risk management for new MCP integrations
    - ~9:10 — Shared configuration in version control: review discipline and quarterly access audit workflow

[^5]: Fireship — "Model Context Protocol Explained in 100 Seconds," YouTube, January 2025. https://www.youtube.com/watch?v=7j_NE6Pjv-E
    - ~0:00 — MCP protocol overview: client-server model, JSON-RPC transport, how Claude Code acts as an MCP client
    - ~0:45 — The three MCP primitives — resources, tools, and prompts — and how each maps to practical integration patterns
    - ~1:20 — Building a minimal custom server in TypeScript using the Anthropic SDK: the minimal viable server scaffold in under 50 lines

[^6]: Anthropic — "Model Context Protocol Introduction," Claude Code Documentation, 2026. https://code.claude.com/docs/en/mcp-introduction
    MCP open standard architecture; the three primitive types (resources, tools, prompts); transport layer options (stdio for local development, SSE for production deployment).

[^7]: David Ondrej — "Build Your Own MCP Server from Scratch (Full Tutorial)," YouTube, February 2026. https://www.youtube.com/watch?v=y4CaCldMDXs
    - ~0:00 — Project setup: initializing the TypeScript MCP SDK and defining the initial server skeleton with resource and tool handler boilerplate
    - ~8:20 — Connecting a custom internal documentation site: search tool implementation, document resource type definition, and authentication against internal APIs using bearer tokens
    - ~22:45 — Deploying to internal infrastructure: health check endpoints, structured error logging, and team credential management without hardcoded secrets
    - ~31:00 — Testing the integration end-to-end: verifying Claude Code can discover and invoke the custom server's tools within a live session

[^8]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
    Audit logging patterns for external service integrations; operational monitoring practices; quarterly review for permission scoping.

[^9]: Jack Herrington — "Claude Code MCP Servers: A Complete Setup Guide," YouTube, November 2025. https://www.youtube.com/watch?v=3QkVZj_nKoA
    - ~12:15 — Security: credential management, minimum-permission scoping, and audit log configuration
    - ~18:40 — Live demo: building and deploying a custom MCP server for internal data access

[^10]: Anthropic — "MCP Security Best Practices," Model Context Protocol Documentation, 2025. https://modelcontextprotocol.io/docs/concepts/security
    Credential management for shared MCP servers; prompt injection risk; the elevated blast radius of compromised shared-server credentials vs. individual credentials.

[^a]: [Tooling: MCP Integration](../Tooling/03-mcp-integration.md) — MCP integration covers the general configuration discipline; this document applies it to custom internal server implementations.
[^b]: [Security: Secrets Management](../Security/04-secrets-management.md) — internal MCP servers often require authentication credentials; secrets management practices apply to MCP server configuration.
[^c]: [MCP Servers: Configuration, Security, and Governance](06-configuration-security-governance.md) — internal server implementations require the same configuration security analysis as commercial MCP servers; the governance framework applies to both.
