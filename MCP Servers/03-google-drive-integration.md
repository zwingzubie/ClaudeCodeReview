## MCP Server 3: Google Drive Integration

**Related to:** [MCP Servers Overview](00-overview.md) — MCP Server 3

---

## Overview

Google Drive is the team's documentation layer — the place where architectural decision records, runbooks, API specifications, onboarding materials, PRDs, and test plans live. It is also entirely invisible to Claude Code sessions. The consequence is that the team maintains two parallel representations of important context: the authoritative Drive documents, and the degraded summaries that appear in CLAUDE.md, prompt boilerplate, and engineer paraphrases. The Google Drive MCP integration eliminates the second representation by making the first one directly accessible.

This memo covers the specific documentation access patterns that deliver value for each role on the team, the folder and naming conventions required to make the integration usable without security risks, the relationship between Drive access and CLAUDE.md maintenance overhead, and the product manager and QA workflows where Drive context access changes session quality most materially.

---

## Section 1: What the Google Drive MCP Server Enables

**Description:** The official Google Drive MCP server provides access to files, folders, Google Docs, Sheets, and Slides stored in Drive, with the ability to search by content and retrieve document text in a form Claude Code can read.[^1] For a team that uses Drive as its documentation layer, the practical effect is that any document stored in a MCP-accessible folder can be referenced directly in a session without the engineer copying its contents into a prompt.

The supported operations include: listing folder contents, retrieving document text, searching by keyword across the accessible folder set, and reading file metadata. Write operations — creating documents, editing content — are available in the API but should not be enabled in the team's initial configuration. The primary value of the Drive integration is read access to existing documentation; write access introduces risks around document integrity that are not justified by the initial use cases.[^2]

Content retrieval handles the Google Docs format natively, returning document text in a structure Claude can read. For Sheets, Claude receives the cell data in a tabular format. For PDFs stored in Drive, text extraction is supported. The integration does not provide access to comments, revision history, or sharing permissions — only the current document content.[^1]

**Recommended Practice:**
- Configure the Google Drive MCP server with read-only OAuth scopes: `drive.readonly` and `drive.metadata.readonly`. Explicitly exclude `drive.file` and `drive` (full access) scopes from the initial configuration.[^1]
- Restrict the integration to specific shared folders rather than granting access to all of Drive. A user's personal Drive contains files that should not be accessible in a team engineering session; access should be scoped to team-shared folders only.[^2]
- Test the content retrieval before rolling out to the team: verify that a known document returns the expected text content, that search returns relevant results for known queries, and that the integration does not surface files from outside the configured folder scope.[^3]
- Review the accessible folder list quarterly: remove folders that are no longer active, add folders for new projects or documentation areas, and audit whether any sensitive documents have been placed in accessible folders inadvertently.[^4]

---

## Section 2: Architecture Documentation and ADRs

**Description:** Architecture decision records are the highest-value Drive content for engineering sessions. An ADR documents a technology choice, the alternatives that were considered, and the reasoning behind the decision. Without ADR access, Claude Code sessions operate in a decision vacuum: Claude proposes approaches based on training data defaults and the current codebase state, without knowing which alternatives the team has already evaluated and rejected. With ADR access, Claude can be instructed to read the relevant ADRs before proposing approaches — producing suggestions that are architecturally consistent rather than accidentally revisiting closed decisions.[^5]

For the architect, this integration shifts the role of ADRs from static archives to active session inputs. An ADR written six months ago that is never read in a session produces no ongoing value; an ADR that Claude reads at the start of every relevant implementation session actively constrains Claude's suggestions to the team's documented architectural intent. The integration is what converts documentation into living constraint.

The same pattern applies to runbooks, system design documents, API specifications, and any other technical document that specifies how the system should behave. These documents currently constrain human behavior in operations; the Drive integration makes them available to constrain AI behavior in sessions.[^6]

**Recommended Practice:**
- Establish a shared Drive folder structure that is explicitly MCP-accessible: `Architecture/ADRs/`, `Architecture/System-Design/`, `Operations/Runbooks/`, `API/Specifications/`. Apply consistent naming conventions so Claude can locate documents by pattern rather than requiring engineers to know the exact file ID.[^2]
- Add a CLAUDE.md instruction that references the ADR folder: when implementing in an area that has relevant ADRs, Claude should search the ADR folder before proposing approaches. This converts ADR-reading from an optional step the engineer might forget into a session default.[^3]
- Instead of copying architectural constraints into CLAUDE.md, reference the Drive folder and instruct Claude to read the relevant documents at session start. This reduces CLAUDE.md maintenance overhead: the ADR is the authoritative source and remains so, rather than requiring a separate update whenever the architectural decision is refined.[^5]
- After each significant architectural decision, create an ADR in the shared folder before the decision is implemented. The integration's value is proportional to the completeness of the ADR corpus; gaps in ADR coverage mean gaps in Claude's architectural context.[^6]

---

## Section 3: Onboarding and Runbook Access

**Description:** Runbooks — step-by-step operational guides for deployments, incident response, environment setup, and recurring maintenance tasks — are a category of documentation where Drive access delivers disproportionate value. Engineers currently either memorize runbook steps, keep them open in a browser tab alongside the Claude Code session, or describe the steps from memory. The Drive integration makes the authoritative runbook directly accessible in the session where it is needed.[^7]

For the QA engineer, Drive access to test plans, test environment setup guides, and test data management runbooks enables sessions that begin with the complete operational context rather than a reconstructed approximation. A test session anchored to the relevant test plan produces coverage that reflects the plan's actual requirements, not what the engineer remembered from the plan.

For onboarding, Drive access to the team's onboarding documentation — environment setup guides, codebase orientation documents, team convention summaries — allows new engineers to use Claude Code as an onboarding guide: Claude reads the relevant section of the onboarding doc and helps the engineer apply it to their specific environment, rather than requiring them to follow static documentation without feedback.[^8]

**Recommended Practice:**
- Organize runbooks in a dedicated `Operations/Runbooks/` folder with consistent naming: `[component]-[operation]-runbook.md` (e.g., `database-migration-runbook.md`, `deployment-staging-runbook.md`). Consistent naming allows Claude to locate runbooks by pattern without requiring engineers to provide file IDs.[^2]
- For operations that currently require engineers to switch between a terminal and a browser-based runbook, use the Drive integration to bring the runbook into the Claude Code session: Claude reads the relevant runbook step, translates it into the engineer's specific environment, and validates the output before proceeding to the next step.[^7]
- Keep runbooks in Drive as the single authoritative source; do not maintain parallel copies in the repository or in CLAUDE.md. When a runbook is updated, the Drive document is updated — and every subsequent session that reads it receives the current version without any additional update step.[^5]
- For new engineer onboarding, instruct Claude to read the onboarding documentation in Drive before beginning any environment setup task. Claude can then provide guidance that is calibrated to the team's actual setup requirements rather than generic environment configuration advice.[^8]

---

## Section 4: Product Manager and QA Workflows

**Description:** The Google Drive integration delivers asymmetric value to the product managers and QA engineer on the team, whose workflows are most constrained by the distance between product documentation (in Drive) and engineering implementation (in Claude Code sessions). Product managers currently relay PRD requirements to engineers verbally or via Linear tickets; the Drive integration allows Claude to read the PRD directly, eliminating the relay and the information loss it introduces.[^9]

For product managers, the integration enables planning sessions that reference the actual product documentation: a session that begins by reading the relevant PRD section before planning sprint work produces a plan anchored to the complete product intent, not a summarized brief. The feature scoping decisions, the explicit non-goals, the acceptance criteria in their full form — all of this is accessible without requiring the product manager to manually import it into the session.

For the QA engineer, acceptance criteria stored in Drive test plan documents can be read directly in test planning sessions. A test plan generated with direct access to the acceptance criteria document is more likely to be complete and correctly scoped than one generated from the engineer's memory of the acceptance criteria. The gap between those two starting points is where test coverage misses originate.[^10]

**Recommended Practice:**
- Establish a shared `Product/PRDs/` folder in Drive, scoped as MCP-accessible, where active PRDs for features in development are stored. The product manager maintains the document; Claude reads it in implementation and planning sessions.[^9]
- For QA sessions, add a session-start convention in CLAUDE.md: when planning tests for a feature, Claude reads the corresponding acceptance criteria document from Drive before generating the test plan. This should be documented as a required step, not optional.[^3]
- Create a lightweight tagging convention for Drive documents that signals their MCP relevance: documents in folders with the prefix `[MCP]` are intended to be read in Claude Code sessions and should be kept current. Documents outside those folders are not guaranteed to be current in the session context.[^2]
- Use Drive document retrieval to close the loop between product and engineering: when a QA session finds behavior that does not match the acceptance criteria, instruct Claude to read the relevant PRD and acceptance criteria document and produce a specific comparison of expected vs. observed behavior. This produces defect reports grounded in the product specification rather than in the engineer's interpretation.[^10]

---

## Section 5: CLAUDE.md Integration and Maintenance Reduction

**Description:** One of the ongoing costs of maintaining CLAUDE.md is that it must be kept synchronized with the authoritative documentation it summarizes. An architectural constraint documented in CLAUDE.md and in an ADR requires updates in two places when the constraint changes. A team convention described in CLAUDE.md and in an onboarding document creates two representations that will diverge over time. The Drive integration resolves this duplication by allowing CLAUDE.md to reference Drive documents rather than summarize them — making Drive the single source of truth and CLAUDE.md a pointer layer rather than a content layer.[^5]

The practical structure is a CLAUDE.md that references Drive folders by convention: "For architectural constraints, read the relevant ADR from `Architecture/ADRs/` before proposing approaches." This instruction tells Claude where to look without embedding the constraint content in CLAUDE.md itself. When an ADR is updated, the CLAUDE.md instruction remains valid; only the Drive document needs updating. The divergence risk is eliminated.

This pattern does require that the referenced Drive documents are well-organized, consistently named, and current. The prerequisite for reducing CLAUDE.md maintenance overhead is investing in Drive document quality — which is a net positive for the team regardless of the Drive integration.[^6]

**Recommended Practice:**
- Audit the current CLAUDE.md for content that exists in a Drive document: architectural patterns that have an ADR, security requirements that have a security runbook, testing requirements that have a test plan template. For each, replace the CLAUDE.md content with a Drive reference and validate that Claude correctly retrieves and applies the Drive document in a session.[^5]
- Establish a monthly CLAUDE.md review practice (see [CLAUDE.md Configuration](../Tooling/01-claude-md-configuration.md)) that specifically checks for Drive document references that may have become stale — Drive documents that have been moved, renamed, or deleted since the CLAUDE.md reference was added.[^3]
- Do not eliminate CLAUDE.md content entirely in favor of Drive references. CLAUDE.md retains value for hard prohibitions, active corrections, and short-form conventions that do not warrant a full document. The principle is: if the content belongs in a Drive document, reference the Drive document; if it belongs only in CLAUDE.md, keep it in CLAUDE.md.[^5]
- When a new Drive document is created that contains session-relevant content, add the Drive reference to CLAUDE.md at the same time. The two documents should evolve together; new Drive documentation that is not referenced from CLAUDE.md is documentation Claude will not access unless explicitly directed to.[^2]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Initial Configuration | Configure read-only scopes; scope to shared architecture and ops folders | Architect |
| ADR Access | Create `Architecture/ADRs/` folder; add CLAUDE.md reference instruction | Architect |
| Runbook Access | Organize `Operations/Runbooks/` with consistent naming conventions | Backend lead |
| Product and QA Workflows | Create `Product/PRDs/` folder; add QA session-start convention to CLAUDE.md | Product manager + QA |
| CLAUDE.md Reduction | Audit CLAUDE.md for Drive-documentable content; replace with references | Architect |

---

[^1]: Google — "Google Drive MCP Server," Google Workspace Developer Documentation, 2025. https://developers.google.com/workspace/mcp
    Official Google Drive MCP server: file and folder access, document content retrieval, content search, permission scoping to specific folders. Authentication via OAuth and service account configuration.

[^2]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    Shared `.mcp.json` project configuration; CLAUDE.md import patterns for referencing external documentation; checking AI configuration artifacts into git as team-owned resources.

[^3]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    CLAUDE.md structure; the relationship between external documentation references and CLAUDE.md content; the import system for modular configuration.

[^4]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
    Audit logging patterns for external service integrations; quarterly review practices for permission scoping.

[^5]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    External tool integration as context engineering; reducing manual context assembly; shared configuration artifacts as team-level productivity multipliers; the Drive integration as a CLAUDE.md maintenance reduction strategy.

[^6]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    Context engineering as the primary discipline in agentic development; how external tool integration shifts workflows from local-only to service-aware sessions.

[^7]: Jack Herrington — "Claude Code MCP Servers: A Complete Setup Guide," YouTube, November 2025. https://www.youtube.com/watch?v=3QkVZj_nKoA
    - ~4:30 — MCP server configuration patterns for documentation access integrations
    - ~12:15 — Security: credential management and minimum-permission scoping for Google Workspace integrations

[^8]: Anthropic — "Model Context Protocol Introduction," Claude Code Documentation, 2026. https://code.claude.com/docs/en/mcp-introduction
    MCP architecture; resource types and how document content retrieval works in the protocol; permission scoping guidance.

[^9]: Greg Kamradt — "MCP Servers for Teams: Governance and Rollout Patterns," YouTube, March 2026. https://www.youtube.com/watch?v=F8pOxXoqFcQ
    - ~0:00 — Team rollout sequencing: why read-only scopes first is risk management
    - ~9:10 — Shared `.mcp.json` in version control: review discipline and quarterly access audits

[^10]: Anthropic — "MCP Security Best Practices," Model Context Protocol Documentation, 2025. https://modelcontextprotocol.io/docs/concepts/security
    Minimum-permission configuration; prompt injection risk via retrieved document content; governance model for write-access MCP operations.
