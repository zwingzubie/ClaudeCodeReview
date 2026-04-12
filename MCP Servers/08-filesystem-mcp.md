## MCP Server 8: Filesystem MCP — Scope Control and Security Implications

**Related to:** [MCP Servers Overview](00-overview.md) — MCP Server 8

---

## Overview

The filesystem MCP server is the most powerful and the most dangerous of the standard MCP integrations. Where other servers extend Claude Code's reach to specific external services — GitHub, Slack, Linear — the filesystem server extends Claude's reach to the local filesystem itself, with the ability to read and write files anywhere in the configured scope. Configured broadly, it can traverse directories, modify files outside the project root, and access configuration and credential files that should never enter a session context. Configured narrowly and deliberately, it is a genuine productivity multiplier for specific workflows that built-in Claude Code tools do not cover.

For a team of 11 working across a complex codebase, the filesystem MCP server's value cases are real but specific: reading local documentation that is not in the repository, accessing shared configuration files in a monorepo root, and reaching files in sibling repositories that share architectural context. These are not hypothetical use cases — they are workflows the backend engineers, architect, and frontend engineers encounter regularly. But each of them requires careful scope definition, because the same capability that enables access to `~/docs/architecture.md` also enables access to `~/.aws/credentials` if the scope is not explicitly bounded.

This memo covers how the filesystem MCP server works, the security implications that are unique to it relative to both other MCP servers and Claude Code's built-in file tools, how to configure scope restrictions that enable the value cases without the risks, the specific workflows where it adds genuine value, and the governance model that keeps path scope bounded over time.

---

## Section 1: What the Filesystem MCP Server Does

**Description:** The built-in filesystem MCP server, maintained as part of the Model Context Protocol reference implementation, exposes the local filesystem as a set of MCP resources that Claude Code can read from and write to within a session.[^1] It operates differently from Claude Code's native file access tools (Read, Edit, Write, Glob, Bash): where the native tools are invoked by Claude during session execution, the filesystem MCP server registers file resources at session startup, making them available throughout the session as named resources rather than requiring explicit tool calls for each access.

The server supports read and write operations. Read operations include listing directory contents, reading file text, and reading file metadata. Write operations include creating files, overwriting file contents, and creating directories. The scope of both read and write operations is bounded by the path configuration in `.mcp.json` — the server will only operate within the configured allowed paths. If no path restriction is configured, the server defaults to the user's home directory, which is an extremely broad default that should never be used in practice.[^2]

For teams that have configured Claude Code's built-in tools for standard project file access, the question is not whether to use the filesystem MCP server instead of native tools but whether the specific workflows that require access outside the project root — to sibling repositories, local documentation directories, or shared monorepo configuration — justify adding the filesystem MCP server with explicit scope configuration for those paths.[^1]

**Recommended Practice:**
- Before configuring the filesystem MCP server, enumerate the specific paths that the use case requires access to. Do not configure the server until there is a concrete path list. A server configured without a specific use case will default to broad scope and sit as an unnecessary permission grant.[^2]
- Treat filesystem MCP as an additive tool, not a replacement for Claude Code's native file access tools. Native tools cover in-project file operations; filesystem MCP covers the specific workflows that require out-of-project access. The distinction should be explicit in CLAUDE.md.[^1]
- Configure the server to read-only for initial deployment. Write access through filesystem MCP should be introduced only when there is a concrete workflow requirement that cannot be served by read-only access plus Claude Code's native write tools.[^3]
- Document the filesystem MCP server's configuration in the MCP server inventory alongside all other servers: what paths are configured, whether read-only or read-write, what use case each path serves, and when the configuration was last reviewed.[^4]

---

## Section 2: Security Implications Unique to Filesystem MCP

**Description:** The filesystem MCP server carries security implications that are qualitatively different from other MCP integrations, for two reasons. First, the filesystem contains a much broader range of sensitive material than any specific external service: SSH keys, AWS credentials, environment files with API keys, browser session data, and application secrets all live on the local filesystem and may be accessible if the server scope is not carefully restricted. Second, the filesystem MCP server can traverse directory structures in ways that other tools cannot — it can follow relative paths, access parent directories of configured roots, and in some configurations reach paths through symlinks that point outside the configured scope.[^2]

The distinction between filesystem MCP and Claude Code's native file tools matters here. Claude Code's native tools operate in the context of the current working directory and the project the engineer is working in; they are oriented toward the project. Filesystem MCP operates in the context of configured paths, which may span multiple projects, personal home directories, or system directories. An overly broad filesystem MCP configuration gives Claude access to a much larger surface area than any individual Claude Code session would typically require.[^3]

Path traversal risk is specific to the filesystem MCP server: a path configured as `/Users/engineer/projects/my-app` that does not explicitly exclude `../` traversal may allow access to `/Users/engineer/projects/` and beyond, depending on the server implementation and version. The mitigation is to configure paths as absolute paths without relying on traversal restrictions, to exclude known sensitive directories explicitly, and to test the scope boundary after configuration by verifying that Claude cannot access files outside the intended paths.[^1]

**Recommended Practice:**
- Never configure the filesystem MCP server with the home directory (`~` or `/Users/engineer`) as the allowed path. The home directory contains credentials, browser data, application secrets, and other sensitive material that should never enter a Claude Code session context.[^2]
- Explicitly exclude `.env` files, credential directories (`~/.aws`, `~/.ssh`, `~/.kube`), and system configuration directories from the filesystem MCP scope. Configure exclusions as explicit patterns in the server configuration rather than relying on Claude's judgment about what to read.[^3]
- After configuring the server, verify the scope boundary by running a test session that attempts to access a file outside the intended scope. If Claude can access it, the scope is wider than configured. Investigate and tighten before deploying to the team.[^1]
- Brief the engineering team on the distinct security posture of filesystem MCP relative to native tools when the server is introduced. Engineers who treat filesystem MCP as equivalent to Claude Code's Read tool in terms of scope are likely to configure it more broadly than they should.[^4]

---

## Section 3: Scoping Configuration

**Description:** The allowed paths configuration in `.mcp.json` is the primary security control for the filesystem MCP server. It must be specific, explicit, and conservative: configured to include exactly the paths the use case requires and nothing beyond. The principle of minimum permission applies with particular force here because the filesystem scope is adjacent to sensitive material in ways that other MCP server scopes are not — broadening the filesystem scope even slightly may bring credential files or environment configurations into reach.[^2]

The recommended configuration pattern specifies absolute paths for each use case, excludes sensitive file patterns explicitly, and documents the rationale for each configured path in a comment block. An allowed path entry with no comment explaining why it is needed and what use case it serves is a configuration artifact that cannot be meaningfully reviewed — it may be correct, or it may be a leftover from a use case that no longer exists.[^1]

For monorepo teams, the filesystem MCP scope is typically the monorepo root and any sibling directories that contain shared configuration. This is a manageable scope if configured with explicit exclusions for credential files and secrets directories. For teams with multiple separate repositories on the same machine, the scope should be the specific directories within each repository that need cross-repository access — not the parent directory containing all repositories.[^3]

**Recommended Practice:**
- Configure allowed paths as an explicit list of absolute paths, one per use case. Do not use wildcard paths or parent directories unless the entire subtree is genuinely needed for the use case. Each path entry should include a comment: `// Architecture docs; read-only; added 2026-01-15 by architect`.[^1]
- Add explicit exclusion patterns for: `.env` and `.env.*` files; `*.pem`, `*.key`, `*.p12` certificate and key files; `~/.aws`, `~/.ssh`, `~/.kube`, and `~/.config` directories; `node_modules` directories (for performance as well as security); and any directory named `secrets` or `credentials`.[^2]
- Review the scope configuration after any change to the team's repository structure, monorepo layout, or documentation storage practices. Filesystem MCP scope configurations become stale as directory structures change, and a stale configuration may be broader than the current use case requires.[^4]
- Require the architect's review for any `.mcp.json` change that adds or expands a filesystem MCP allowed path, in addition to the standard CODEOWNERS review. Filesystem path additions carry more security surface than other MCP configuration changes and warrant additional scrutiny.[^3]

---

## Section 4: Use Cases Where Filesystem MCP Adds Value

**Description:** The value cases for filesystem MCP are specific and bounded. They are not served adequately by Claude Code's native file tools, and they are common enough in the team's workflows to justify the governance overhead of maintaining a correctly scoped server configuration. The three primary value cases are local documentation access, shared monorepo configuration access, and cross-repository architecture context.[^3]

Local documentation access covers situations where the team maintains a local `~/docs` directory or equivalent containing architecture notes, runbooks, API references, or vendor documentation that is not committed to the repository. When an engineer is working on a feature that requires consulting these documents, the filesystem MCP server allows Claude to read them directly in the session rather than requiring the engineer to copy relevant sections into the prompt. This is a genuine productivity gain, and it is a read-only use case that carries low risk when configured with an explicit allowed path for the documentation directory.[^1]

Shared monorepo configuration is relevant for teams whose repository structure includes shared tooling configuration, ESLint rules, TypeScript base configs, or environment configuration templates at the monorepo root or in a shared package that is not always in the current working directory of a feature session. Filesystem MCP allows Claude to read these configs in context without the engineer needing to navigate to them and copy their contents.[^2]

Cross-repository architecture context applies when the frontend and backend repositories share data models, API contracts, or type definitions that exist as source files in the sibling repository. A backend session that needs to validate API response shapes against the frontend's expected types benefits from direct access to the frontend type files rather than relying on the engineer's summary.[^4]

**Recommended Practice:**
- Configure a dedicated read-only allowed path for the team's local documentation directory if one exists. Scope it to the specific directory (e.g., `/Users/engineer/team-docs`), not to the parent directory. Document the path and use case in `.mcp.json` and the MCP inventory.[^1]
- For monorepo teams, configure the monorepo root as a read-only allowed path with explicit exclusions for node_modules, `.env` files, and credential directories. Test the configuration by verifying that Claude can read shared configuration files but cannot read `.env` at the root.[^2]
- For cross-repository type sharing, configure the specific path within the sibling repository that contains the shared types (e.g., `/Users/engineer/projects/frontend/src/types`) rather than the entire sibling repository root. Narrow paths are always preferable to broad paths.[^3]
- Document the active use cases for filesystem MCP in CLAUDE.md so engineers know when to use it versus native tools. Engineers who do not know why the server is configured may either ignore it (failing to get value) or use it unnecessarily (expanding session scope beyond what the task requires).[^4]

---

## Section 5: Governance

**Description:** Filesystem MCP governance requires a more active maintenance posture than other MCP servers because the filesystem is not static — engineers add new credential files, create new sensitive directories, and change repository structures in ways that may inadvertently bring new sensitive material into scope. A path that was safe when configured six months ago may now contain a secrets subdirectory that was added since the last review. The quarterly path audit is not optional; it is the mechanism that keeps the scope accurate as the environment changes.[^3]

The architect's approval requirement for filesystem MCP scope changes serves a different function than for other MCP servers. For GitHub or Slack, the architect reviews scope changes to prevent excessive permission grants to external services. For filesystem MCP, the architect reviews scope changes to prevent inadvertent exposure of sensitive local files that may not be visible in the diff of the `.mcp.json` change itself. The architect needs to understand not just what path is being added but what the filesystem structure at that path currently contains.[^2]

Production server filesystem access is categorically prohibited. Even in a development context, a filesystem MCP configuration that includes a path to a production server's mounted filesystem — or to a directory that contains production credentials — violates the governance principle that Claude Code sessions never interact with production data or credentials. The prohibition applies regardless of whether the path is technically accessible from the development machine.[^1]

**Recommended Practice:**
- Conduct a quarterly path audit: for each configured filesystem MCP allowed path, verify that the path still exists, still contains only what it was configured to access, has not had new credential or secret files added since the last audit, and is still actively used by at least one team member's workflow. Remove paths that fail any criterion.[^3]
- Require architect review for any allowed path addition or expansion in `.mcp.json`. The review should include a brief statement of what the directory currently contains and confirmation that no credential or secret files are present or will be present at that path.[^2]
- Prohibit filesystem MCP configuration on any machine that has mounted filesystems from production servers or direct filesystem access to production environments. Document this prohibition in the engineering handbook alongside the MCP server setup guide.[^1]
- When a team member's local filesystem structure changes — a new repository is cloned, a shared drive is mounted, a documentation directory is reorganized — review the filesystem MCP configuration to confirm that the change has not brought new material into scope or made an existing allowed path outdated.[^4]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Initial Configuration | Enumerate use cases and specific paths before configuring; add to MCP inventory | Architect |
| Security Scope | Exclude .env, credentials directories, and system configs explicitly; test scope boundary | Architect |
| Path Configuration | Use absolute paths with per-path comments; require architect review for additions | Architect |
| Use Case Documentation | Document active filesystem MCP use cases in CLAUDE.md | Backend lead |
| Governance | Conduct quarterly path audit; prohibit production filesystem access; rotate credentials on schedule | Architect |

---

[^1]: Model Context Protocol — "Server: Filesystem," modelcontextprotocol.io, 2025. https://modelcontextprotocol.io/docs/servers/filesystem
    Official filesystem MCP server documentation: allowed paths configuration, read/write operation support, default scope behavior, and exclusion pattern configuration. Comparison with Claude Code's native file access tools.

[^2]: Anthropic — "MCP Security Best Practices," Model Context Protocol Documentation, 2025. https://modelcontextprotocol.io/docs/concepts/security
    Minimum-permission configuration; path traversal risk in filesystem MCP; credential exposure vectors; the governance model for filesystem access grants in team environments.

[^3]: Greg Kamradt — "MCP Servers for Teams: Governance and Rollout Patterns," YouTube, March 2026. https://www.youtube.com/watch?v=F8pOxXoqFcQ
    - ~9:10 — Filesystem MCP scope configuration: how broad defaults create credential exposure risk; absolute path specification as the primary mitigation
    - ~17:50 — Quarterly path audit: the discipline that prevents filesystem scope drift as directory structures change over time

[^4]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    Shared `.mcp.json` project configuration; CLAUDE.md documentation patterns for tool-specific use cases; the governance artifact standard for external integrations including filesystem access.

[^5]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    Local documentation access as a context engineering pattern; filesystem MCP as the integration that enables session-local documentation context without manual copy-paste; the cross-repository context use case for distributed frontend/backend teams.
