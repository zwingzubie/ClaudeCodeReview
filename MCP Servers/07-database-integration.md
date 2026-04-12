## MCP Server 7: Database Integration (Postgres, SQLite, and Schema Access)

**Related to:** [MCP Servers Overview](00-overview.md) — MCP Server 7

---

## Overview

Database MCP access closes one of the most persistent gaps in AI-assisted backend development: the gap between the schema Claude knows about and the schema that actually exists in the database. When backend engineers describe their schema to Claude through text or paste in migration files, they introduce summarization errors, omit recent changes, and create a session environment that diverges from the codebase's ground truth. The Postgres MCP server eliminates that gap by giving Claude direct schema introspection capability — the ability to inspect live table definitions, column types, constraints, and indexes before writing a single line of code.

For a team of four backend engineers working on a codebase with an actively evolving schema, the value is compounding. Every migration session that starts from the correct schema rather than a remembered or paraphrased version produces migrations that are consistent with what exists, not with what the engineer thought existed at the time of prompting. Every query Claude writes in a session with live schema access reflects actual column names, actual types, and actual foreign key relationships — eliminating a category of errors that otherwise surfaces in review or at runtime.

This memo covers the operational case for database MCP access, the configuration of the Postgres MCP server, safe query patterns that prevent unintended writes and DDL operations, schema-aware migration workflows, and the governance constraints that apply to database connections in any Claude Code session. The common thread is precision: database access in Claude sessions is most valuable when it is exact, most dangerous when it is broad, and most sustainable when it is governed from the first connection rather than retrofitted after an incident.

---

## Section 1: Why Database MCP Access Is High-Value

**Description:** Schema drift is the condition in which the database schema that exists in production or staging diverges from what is documented, remembered, or described in a session. It is endemic in active codebases: migrations run, columns are renamed, new constraints are added, and the session-start description of the schema falls behind the reality. Claude Code sessions that rely on paraphrased schema descriptions amplify this problem — Claude writes queries and migrations calibrated to the schema as described, not as it exists, and the discrepancy surfaces at the worst possible time: during review, in test runs, or in production.[^1]

Direct schema introspection via the Postgres MCP server addresses this structurally. A session that begins by running `\d table_name` or equivalent introspection queries has ground-truth schema context before any code is written. The session can identify columns that exist but were not mentioned in the task description, constraints that rule out a proposed approach, and indexes that should influence query design. This is qualitatively different from a session working from a summarized schema — the introspection is exact where summarization is approximate.[^2]

For the team's four backend engineers, the practical value concentration is in three workflows: writing complex queries against multi-table schemas, writing migrations that must be consistent with existing table definitions, and debugging query performance by understanding existing indexes. All three benefit directly from live schema access, and all three are workflows that the backend engineers perform frequently enough that the cumulative value of reducing schema-drift errors is material.[^3]

**Recommended Practice:**
- Configure the Postgres MCP server as a standard tool in the shared `.mcp.json` for all backend engineers. Include a comment block specifying that the connection uses the development or staging database, not production, and that the connection string is read-only.[^1]
- Establish a session-start convention for backend sessions involving database work: Claude reads the relevant table schemas before writing any query or migration. Document this in CLAUDE.md under a "Database Sessions" heading.[^2]
- Measure schema-drift errors before and after the MCP integration: track incidents where a migration or query failed because it referenced a column or constraint that did not match the actual schema. This data supports the investment in the integration and calibrates expectations for new team members.[^3]
- For engineers who are new to the codebase's schema, database MCP access is especially high-value: a session that introspects the schema before asking questions about it produces better questions and avoids common misconceptions about table relationships that otherwise require correction mid-session.[^4]

---

## Section 2: Postgres MCP Server Setup

**Description:** The official `@modelcontextprotocol/server-postgres` package, maintained as part of the Model Context Protocol reference server suite, provides Postgres connectivity for Claude Code sessions.[^5] It exposes three primary capabilities: schema introspection (listing tables, describing table structures, listing indexes and constraints), query execution (running SELECT queries and returning results), and database metadata access (listing schemas, databases, and Postgres version information). The server does not support DDL operations by default and should be configured to run against a read-only connection string.

The connection string configuration is the most critical security decision in the setup. The connection string must use a database user with SELECT privileges only — not the application user, not a superuser, and never a production connection string. The development or staging database is the appropriate target; the schema is identical to production for the purpose of session introspection, but the data is not production data and the access path does not touch the production environment.[^2]

For teams using SQLite rather than Postgres, `@modelcontextprotocol/server-sqlite` provides equivalent functionality with filesystem-based databases. The security considerations differ — the SQLite file is accessed directly — but the session-start introspection convention and the read-only access principle apply equally.

**Recommended Practice:**
- Install and configure `@modelcontextprotocol/server-postgres` with a dedicated read-only database user. The user should have `CONNECT` and `SELECT` privileges on the development/staging database. Do not reuse the application's runtime database user; a dedicated introspection user makes the permission boundary explicit and auditable.[^5]
- Store the connection string in environment variables (e.g., `MCP_POSTGRES_URL`), never inline in `.mcp.json`. Reference the variable from the configuration file. This keeps the connection string out of version control while keeping the configuration itself committable.[^1]
- Configure the server with `allowedOperations: ["schema", "select"]` if the server version supports operation scoping. If not, use a read-only database user as the enforcement mechanism. Document which layer enforces the read-only constraint in the `.mcp.json` comment block.[^2]
- Test the connection before team rollout by running a session that introspects a known table and verifies the output matches the migration history. Validate that the read-only user cannot execute INSERT, UPDATE, DELETE, or any DDL operations — and log the validation result in the MCP server inventory.[^3]

---

## Section 3: Safe Query Patterns

**Description:** Database MCP access in Claude Code sessions introduces a query execution surface that requires explicit safety discipline. The MCP server can run queries that return large result sets, execute long-running operations, or — if misconfigured — execute write operations against the database. The safe query patterns that apply to any database interaction apply here, but with particular emphasis on the session context: Claude may write queries that seem correct given its schema understanding but produce unexpected results or performance characteristics against the actual data.[^2]

The four safe query patterns for MCP database sessions are: read-only enforcement (connection user has no write privileges), result set limits (all queries include explicit LIMIT clauses to prevent unbounded result sets), no DDL operations (CREATE, ALTER, DROP, TRUNCATE are not executed in any MCP session — schema changes happen exclusively through migration files), and parameterized query patterns (when Claude writes queries for production use, they use parameterized form; MCP session queries for inspection use literal values but are not the source of production query strings).[^1]

The boundary between schema introspection and query execution is also a risk boundary. Introspection is low-risk: it reads metadata, produces no side effects, and returns bounded result sets. Query execution is higher-risk: it reads data, may produce large result sets if not bounded, and operates on actual records. Sessions that require only schema context for migration writing should restrict the server to introspection operations; only sessions explicitly performing query design and optimization should enable query execution.[^3]

**Recommended Practice:**
- Configure Claude Code sessions involving database MCP to append `LIMIT 100` to all SELECT queries unless the engineer explicitly overrides it. Document this convention in CLAUDE.md. The convention prevents accidental large result set retrieval that could expose sensitive data or slow the session.[^2]
- Establish a clear rule in CLAUDE.md: no DDL operations are ever executed through the MCP server. Schema changes are written as migration files, reviewed, and applied through the team's standard migration workflow. Claude uses the MCP server to read schema state, not to change it.[^1]
- For sessions focused on migration writing, restrict the MCP connection to schema introspection only by configuring a database user with no SELECT privilege on data tables — only information_schema and pg_catalog access. This limits the session's database access to exactly what migration writing requires.[^5]
- Log all query executions from MCP sessions using the database's query log. Review the log in the quarterly MCP audit to confirm that only SELECT and introspection operations appear. Any write operation in the log is a configuration incident requiring immediate investigation.[^4]

---

## Section 4: Schema Access for Migration Sessions

**Description:** Migration writing is the use case that delivers the most consistent, measurable value from database MCP access. A migration session that begins with schema introspection — Claude reads the current table definitions, column types, constraints, and indexes before writing the migration — produces migrations that are consistent with what exists. A migration session without schema access produces migrations that are consistent with what the engineer remembers about the schema, which is a different and less reliable thing.[^3]

The practical workflow is: before writing any migration, Claude introspects the tables the migration will touch and produces a summary of the current state. The engineer reviews the summary against their expectations — if there is a discrepancy, it is surfaced before the migration is written, not after it fails to apply. Claude then writes the migration with full knowledge of the current column names, types, and constraints, and can validate that the migration's operations are consistent with the existing schema (e.g., that a column being renamed actually exists, that a foreign key constraint references a table that exists with the expected primary key type).[^2]

Post-migration validation is the second step: after writing the migration, Claude can verify that the resulting schema (current state plus migration operations) is consistent with the application code that will rely on it. This requires Claude to read both the migration and the relevant ORM models or query code to check for mismatches. The database MCP server provides the current state; the standard file access tools provide the application code; together they give Claude the context needed for end-to-end migration validation.[^1]

**Recommended Practice:**
- Establish a migration session template in CLAUDE.md: (1) introspect tables to be modified, (2) summarize current schema state, (3) engineer confirms or corrects the summary, (4) write migration, (5) validate migration against ORM models or query code. Every migration session follows this sequence.[^2]
- After writing a migration, have Claude generate a pre-apply checklist: column names referenced in the migration match what exists in the current schema; foreign key references match table and column names that currently exist; index names do not conflict with existing indexes; rollback operations are the exact inverse of apply operations.[^3]
- For complex migrations involving multiple table changes, run introspection on each affected table before beginning rather than retrieving schema context on demand mid-session. Complete schema context at session start produces more coherent migrations than piecemeal context retrieval.[^1]
- Track migration failures attributable to schema mismatch before and after implementing the schema-access-first convention. A reduction in schema-mismatch migration failures is the primary success metric for the database MCP integration in migration workflows.[^4]

---

## Section 5: Governance

**Description:** Database connections in Claude Code sessions carry governance obligations that do not apply to other MCP integrations. A Slack or Linear connection's blast radius is limited to those services' data. A database connection's blast radius extends to every record in the database — which may include customer data, financial records, credentials, or other sensitive information. The governance model for database MCP access must be proportional to this elevated risk profile.[^4]

The non-negotiable governance constraints are: production credentials are never used in any Claude Code session, development or staging databases are the only permissible targets, all queries are logged at the database level, and any grant of write access requires architect approval and a documented rationale. These constraints are not bureaucratic — they are the minimum controls that distinguish responsible database MCP adoption from a data exposure risk.[^1]

The architect's role in database MCP governance is more active than in other MCP integrations. Write-access grants require architect approval specifically because a write access misconfiguration — Claude executing a DELETE or UPDATE in a session — cannot always be reversed cleanly. The architect is the escalation point for any engineer who believes a use case requires write access, and the architect's approval documentation should specify the scope, duration, and monitoring requirements of any write-access grant.[^5]

**Recommended Practice:**
- Document a standing rule in CLAUDE.md and the MCP server inventory: production database credentials are never configured in any MCP server or Claude Code session context, under any circumstances. Violations are treated as security incidents, not policy misunderstandings.[^4]
- Enable query logging on the development/staging database instance used for MCP connections. Log entries should capture the query text, the database user, and the timestamp. Include a review of these logs as a standing item in the quarterly MCP governance audit.[^1]
- Require architect approval before granting any write access to a database MCP connection. The approval must document the use case, the tables to be written, the duration of the write-access grant, and the monitoring arrangement that will detect unintended writes.[^5]
- Rotate the MCP database user's credentials on the team's standard secret rotation schedule — at minimum annually, and immediately following any session where the connection string may have been exposed (e.g., accidentally printed in a session transcript). Document the rotation dates in the MCP server inventory.[^2]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| MCP Server Setup | Install server-postgres with dedicated read-only user; store connection string in env var; test and validate | Backend lead |
| Schema-First Convention | Add database session template to CLAUDE.md; enforce introspection before coding | Backend lead |
| Safe Query Patterns | Configure LIMIT convention; add no-DDL rule to CLAUDE.md; enable query logging | Backend lead |
| Migration Workflow | Implement five-step migration session template; add pre-apply checklist | Backend lead |
| Governance | No-production-credentials rule in CLAUDE.md; architect approval required for write access | Architect |

---

[^1]: Anthropic — "Model Context Protocol Introduction," Claude Code Documentation, 2026. https://code.claude.com/docs/en/mcp-introduction
    MCP architecture overview; permission scoping guidance; read-only enforcement patterns; server configuration in .mcp.json with environment variable credential referencing.

[^2]: Model Context Protocol — "Server: Postgres," modelcontextprotocol.io, 2025. https://modelcontextprotocol.io/docs/servers/postgres
    Official Postgres MCP server documentation: installation, connection string configuration, supported operations (schema introspection, SELECT execution), and operation scoping. Read-only enforcement at the database user level.

[^3]: Model Context Protocol — "Server: SQLite," modelcontextprotocol.io, 2025. https://modelcontextprotocol.io/docs/servers/sqlite
    SQLite MCP server documentation and comparison with Postgres server: schema introspection API, query execution patterns, and the information_schema access pattern for metadata-only sessions.

[^4]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    Session context discipline; credential security practices; CLAUDE.md convention patterns for role-specific workflows; the governance documentation standard for external integrations.

[^5]: Anthropic — "MCP Security Best Practices," Model Context Protocol Documentation, 2025. https://modelcontextprotocol.io/docs/concepts/security
    Minimum-permission configuration; credential management for MCP server connections; the three primary MCP security risks applied to database integration contexts; write-access governance model.

[^6]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    External tool integration as context engineering; how live data access replaces summarization in high-value backend sessions; schema drift as a systemic error source in AI-assisted database development.
