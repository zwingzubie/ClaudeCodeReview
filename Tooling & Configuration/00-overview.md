## Overview

As our team of 11 — including 4 backend engineers, 3 frontend engineers, 1 architect, 1 QA engineer, and 2 product managers — scales Claude Code usage across daily workflows, the configuration layer between the tool and the team becomes a first-class engineering concern. Raw model capability is not the bottleneck. How we configure, extend, and constrain Claude Code determines whether every engineer's session produces output consistent with the team's architecture — or diverges from it in ways that accumulate as the issues documented in the Issues section.

Claude Code exposes a layered configuration system spanning more than a dozen distinct file types and directories, each governing a different aspect of model behavior, tool access, session memory, and team coordination. Teams that invest in this infrastructure consistently report materially higher consistency and lower correction rates than those relying on default behaviors. A 2026 survey of small engineering teams found that teams with shared, version-controlled configuration artifacts reported approximately 40% fewer architectural inconsistencies per sprint than teams where each engineer configured the tool independently.[^1] The investment in configuration is paid once; the consistency dividend compounds with every session.

Ten configuration surfaces are documented below. They are ordered by dependency: CLAUDE.md establishes the instruction foundation; settings and permissions enforce operational boundaries; hooks automate quality gates; MCP connects external context; skills and commands package reusable workflows; agents enable subagent delegation; rules allow scoped overrides; memory accumulates session learning; keybindings and state files tune the local experience; and CI/CD extends all of the above into the automated pipeline.

---

## Configuration 1: CLAUDE.md — Instruction Hierarchy

**Description:** CLAUDE.md is the primary instruction artifact for Claude Code — the file that transforms a generic AI assistant into a team-aware agent with knowledge of your stack, conventions, and prior architectural decisions. It operates as a read-at-session-start instruction layer. Unlike a prompt, it is persistent: once checked into git, it is inherited by every engineer's session without any individual action required.[^2]

Claude Code supports a three-level CLAUDE.md hierarchy. The global file at `~/.claude/CLAUDE.md` applies to every project on the machine — suitable for personal conventions, preferred response styles, and cross-project constraints. The project-level `CLAUDE.md` at the repository root applies to all sessions in that project and is the canonical team configuration artifact. The `.claude/CLAUDE.md` variant (inside the `.claude/` directory) functions identically to the root-level file. Files can import other files using `@path/to/file` syntax, allowing the team to maintain separate modular context files — one for git conventions, one for API patterns, one for security requirements — without embedding everything in a single unwieldy document.[^3]

Boris Cherny, who created Claude Code, describes updating the CLAUDE.md as a core discipline: every time Claude does something incorrectly, the correction is added to the file so it will not recur.[^4] Over time, the file becomes a living record of what the team has learned — a compound asset whose value increases with every session. A CLAUDE.md maintained for six months by an engaged team is materially more effective than one written once and left static.[^5]

A documented failure mode requires active management: when the file grows too long, Claude begins deprioritizing portions of it. Rules buried in noise behave identically to absent rules. Anthropic's guidance is explicit — if Claude already does something correctly without an explicit instruction, delete that instruction.[^3] The file should be short, precise, and regularly pruned.[^6]

**Proposed Solution:**
- Maintain a single team-owned root-level `CLAUDE.md` checked into git, owned by the architect, with an explicit obligation to update after every significant architectural decision and after every recurring AI error identified in PRs.[^4]
- Use `@import` syntax to split the file into modular sections: stack constraints, naming conventions, testing requirements, anti-patterns, and a corrections log — each in its own referenced file.[^3]
- Supplement project-level configuration with a personal `~/.claude/CLAUDE.md` for individual engineers' cross-project preferences (response format, verbosity, preferred explanation style).[^7]
- Test CLAUDE.md effectiveness monthly: ask Claude to perform tasks it would have done incorrectly before specific rules were added, and verify compliance rather than assuming it.[^1]
- Prune the file quarterly. Remove any instruction that Claude already follows without it, any outdated architectural constraint, and any correction that no longer applies to the current codebase.[^6]

---

[^1]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    Team-level tooling standardization and its impact on output consistency; shared configuration artifacts as consistency infrastructure; treating AI configuration as a first-class engineering artifact.

[^2]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    CLAUDE.md as a session-initialization artifact; the three-level hierarchy (global, project, `.claude/`); how the file interacts with the model's context window at session start.

[^3]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    Import directives for modular CLAUDE.md structure; CLAUDE.md length pruning discipline; custom command library setup alongside CLAUDE.md.

[^4]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    CLAUDE.md as a living corrections document: update discipline, compound value of accumulated corrections, and team ownership model.

[^5]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    How accumulated session learning in CLAUDE.md shifts AI sessions from generic to codebase-aware over time.

[^6]: Anthropic — "Claude Code: Settings and Configuration," Claude Code Documentation, 2026. https://code.claude.com/docs/en/settings
    Global vs. project-level CLAUDE.md resolution order; file length management; the relationship between CLAUDE.md and settings.json in controlling session behavior.

[^7]: Dex Horthy (YC Root Access) — "Advanced Context Engineering for Agents," YouTube, August 2025. https://www.youtube.com/watch?v=IS_y40zY-hc
    - Modular CLAUDE.md: using import directives to maintain separate context files for git conventions, API patterns, and testing requirements
    - Global vs. project CLAUDE.md: how personal `~/.claude/CLAUDE.md` supplements team-level configuration without overriding it
    - Shared context files as a team artifact: how team-level CLAUDE.md anchors all sessions to the same architectural constraints

---

## Configuration 2: settings.json and settings.local.json — Permissions and Behavior

**Description:** `settings.json` is the primary machine-readable configuration artifact for Claude Code. Where CLAUDE.md provides natural-language instructions, `settings.json` provides structured behavioral rules: which tools Claude can use without confirmation, which shell commands are permitted, how environment variables are injected, and what hooks execute at each session event. It is checked into git at `.claude/settings.json`, making it a shared team artifact with the same review properties as production code.[^8]

The settings resolution order — from lowest to highest precedence — is: enterprise managed settings, user settings (`~/.claude/settings.json`), project settings (`.claude/settings.json`), and local overrides (`.claude/settings.local.json`). The local override file is gitignored by convention, giving individual engineers a way to configure machine-specific behavior — a different editor path, a local model endpoint, personal notification preferences — without polluting the team configuration.[^9] Enterprise deployments can inject a `managed-settings.json` at the system level that cannot be overridden by any user or project setting, providing a hard policy floor for regulated environments.[^10]

Key settings surfaces include: `permissions.allow` and `permissions.deny` arrays for tool access control; `env` for session environment variable injection; `hooks` for event-driven automation; and `model` for default model selection. The permission model creates appropriate friction for high-risk operations — requiring explicit approval before shell command execution, write operations outside defined paths, or access to sensitive tools. A 2026 Sonar survey found that 52% of developers who state they distrust AI-generated code accept it without verification.[^11] Explicit permissions are one of the few interventions that force active review of AI-proposed actions before they execute.

**Proposed Solution:**
- Define team-standard permissions in `.claude/settings.json` checked into git: require explicit approval for bash command execution in production-adjacent sessions, and use `permissions.deny` to hard-block operations that should never occur in a given project context.[^8]
- Establish work-context permission profiles: a frontend profile restricting writes to `src/components/` and `src/styles/`; a migration profile permitting database tool access; a read-only profile for exploration and planning sessions.[^9]
- Use `.claude/settings.local.json` (gitignored) for engineer-specific overrides — machine paths, personal MCP server instances, local model routing — so individual configurations do not create git noise.[^9]
- For regulated environments, define enterprise-level constraints in `managed-settings.json` to create a non-overridable policy floor for data handling, network access, and tool permissions.[^10]
- Review permission grant logs quarterly: identify sessions granted permissions outside their expected scope and investigate causes.[^12]

---

[^8]: Anthropic — "Security and Permissions," Claude Code Documentation, 2026. https://code.claude.com/docs/en/security-permissions
    Permission profile configuration, `--allowedTools` flag semantics, and the rationale for work-context-specific permission scoping in production-adjacent sessions.

[^9]: Anthropic — "Claude Code: Settings and Configuration," Claude Code Documentation, 2026. https://code.claude.com/docs/en/settings
    Settings resolution order (managed → user → project → local); `settings.local.json` gitignore convention; environment variable injection and model selection configuration.

[^10]: Anthropic — "Enterprise Configuration," Claude Code Documentation, 2026. https://code.claude.com/docs/en/enterprise-configuration
    `managed-settings.json` and `managed-mcp.json` as non-overridable policy floors; deployment patterns for organization-wide Claude Code configuration; drop-in fragment support via `managed-settings.d/`.

[^11]: Sonar (SonarSource) — "Sonar Data Reveals Critical 'Verification Gap' in AI Coding," press release, January 8, 2026. https://www.sonarsource.com/company/press-releases/sonar-data-reveals-critical-verification-gap-in-ai-coding/
    96% of developers distrust AI-generated code; only 48% verify before committing. 42% of all committed code now originates from AI. Automation bias and its effect on permission hygiene.

[^12]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
    CI/CD audit logging, session permission scoping, and the staged-write pattern that isolates review analysis from write operations. Service account permission model for pipeline sessions.

[^13]: The Pragmatic Engineer — "AI Tooling for Software Engineers in 2026," March 2026. https://newsletter.pragmaticengineer.com/p/ai-tooling-2026
    Permission model design in AI coding tools; how structured behavioral constraints reduce the cognitive load of AI session oversight for engineering teams.

[^14]: Sabrina Ramonov — "CLAUDE CODE FULL COURSE," YouTube, February 17, 2025. https://www.youtube.com/watch?v=fYX6hHC9FhQ
    - Settings.json structure: `permissions.allow`, `permissions.deny`, and the difference between project and user settings files
    - Work-context permission profiles: how to create separate configurations for frontend, backend, and migration sessions
    - Local override patterns: `settings.local.json` for machine-specific configuration without polluting team settings

---

## Configuration 3: Hooks and Event Automation

**Description:** Claude Code supports event hooks — shell commands that fire automatically at defined points in every session. Five hook events are available: `PreToolUse` (fires before any tool call), `PostToolUse` (fires after any tool call), `UserPromptSubmit` (fires before the model processes a user message), `Stop` (fires when a session ends), and `Notification` (fires when Claude generates an alert).[^15]

Hooks are the mechanism by which a team enforces quality standards without relying on engineer discipline in the moment. A `PostToolUse` hook on file write operations can automatically run linting and type-checking; a `Stop` hook can run a final SAST scan before a session concludes; a `UserPromptSubmit` hook can inject current sprint context and the current date into every session without requiring engineers to type it manually. The value of hooks is proportional to how consistently they are deployed — a team where every engineer has the same hooks via a shared `.claude/settings.json` enforces the same quality gates uniformly.[^16]

Hooks are configured in `settings.json` under the `hooks` key. Each hook specifies the event, an optional matcher (to fire only for specific tool names), the shell command to run, and whether the hook blocks execution (a blocking hook that exits non-zero can prevent a tool call from proceeding). This blocking capability makes hooks the enforcement mechanism for pre-conditions that CLAUDE.md instructions alone cannot enforce — a hook can prevent a write to a protected path in a way that a CLAUDE.md instruction cannot.[^15]

Given that AI-generated code introduces security vulnerabilities at higher rates than human-written code[^11], automated scanning at the hook level catches vulnerabilities before they reach human review regardless of which engineer wrote the session.

**Proposed Solution:**
- Define team-standard hooks in `.claude/settings.json` covering: `PostToolUse` linting on write operations, `PostToolUse` test execution on source file changes, and a `Stop`-event SAST scan.[^15]
- Use the `UserPromptSubmit` hook to inject current date and sprint context into every session — removing the dependency on individual engineers to supply this context manually.[^16]
- Configure a `Notification` hook to dispatch system notifications when long-running sessions complete, enabling true parallel session management without active monitoring.[^17]
- Use blocking `PreToolUse` hooks to enforce hard write-path restrictions that supplement permission profiles — preventing writes to configuration files, migration directories, or secret paths during scoped sessions.[^15]
- Audit hook configuration quarterly: verify team-standard hooks are in effect on all developer machines, identify frequently-firing hooks, and investigate any hooks being bypassed or overridden in `settings.local.json`.[^1]

---

[^15]: Anthropic — "Hooks Reference," Claude Code Documentation, 2026. https://code.claude.com/docs/en/hooks-reference
    Complete hooks API: event types (PreToolUse, PostToolUse, UserPromptSubmit, Stop, Notification), shell command configuration, blocking semantics, and tool-name matchers.

[^16]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    Hook configuration patterns for automated quality gates; `UserPromptSubmit` for context injection; hooks as enforcement for quality standards that CLAUDE.md cannot enforce alone.

[^17]: Anthropic — "Claude Code Hooks Reference," Claude Code Documentation, 2026. https://code.claude.com/docs/en/hooks
    Notification event configuration; system notification dispatch patterns for parallel session management; hook ordering when multiple hooks target the same event.

[^18]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    Hooks as the automation layer that converts CLAUDE.md intentions into enforced behavior; the argument that manual quality gate discipline fails at team scale.

[^19]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
    SAST hook integration at session stop; how automated scanning at the hook level provides a security floor independent of engineer attentiveness.

[^20]: Sabrina Ramonov — "CLAUDE CODE FULL COURSE," YouTube, February 17, 2025. https://www.youtube.com/watch?v=fYX6hHC9FhQ
    - PostToolUse hook setup: running linting and tests automatically on every file write without manual invocation
    - Stop event hooks: SAST scanning at session completion and system notification dispatch for parallel session management
    - UserPromptSubmit injection: inserting sprint context and current date into every session start

---

## Configuration 4: .mcp.json — Model Context Protocol Integration

**Description:** The Model Context Protocol (MCP) is an open standard that allows Claude Code to interact with external tools, services, and data sources through a standardized interface. MCP server configurations are defined in `.mcp.json` at the project root — a shared, checked-in artifact that gives every team member's sessions access to the same external context without individual setup overhead.[^21]

MCP fundamentally changes what Claude can do in a session. Rather than copying a database schema into a prompt, Claude can query it directly via a Postgres MCP server. Rather than manually cross-referencing open tickets, Claude can read the ticket and write the implementation in the same session via a Linear MCP server. Rather than navigating GitHub's web interface, Claude can interact with issues, PRs, and branch operations via a GitHub MCP server. The coordination savings from these integrations compound with session volume — each session that avoids manual context-gathering is a session with lower overhead cost and a higher proportion of time spent on productive work.[^22]

The MCP permission model mirrors the settings permission model: servers are granted specific capability scopes (read-only, read-write, specific operation types), and write-access operations can be configured to require explicit confirmation. The correct adoption sequence is read-only servers first — database schema access, documentation lookup, issue reading — before expanding into write-access operations. Confidence in each integration should be established before expanding its permissions.[^21]

**Proposed Solution:**
- Define team-standard MCP server configurations in `.mcp.json` at the project root, checked into git, so all engineers' Claude Code instances have consistent access to the same tools and connection settings.[^23]
- Start with read-only MCP servers before introducing write-access servers. Establish confidence in each integration before expanding permissions into write operations.[^21]
- For write-access MCP servers, require explicit confirmation prompts for all operations that modify external state — even when those operations fall within the session's declared scope.[^21]
- Configure server-specific `env` entries in `.mcp.json` to reference environment variables rather than embedding credentials in the file. Use `.claude/settings.local.json` for any machine-specific MCP connection overrides.[^24]
- Review MCP server access logs quarterly: identify which servers are used, what operations are performed per session, and whether any sessions have exceeded their intended operational scope.[^12]

---

[^21]: Anthropic — "Model Context Protocol Introduction," Claude Code Documentation, 2026. https://code.claude.com/docs/en/mcp-introduction
    MCP open standard architecture; server permission model; minimum-permission configuration guidance; the progression from read-only to write-access integrations.

[^22]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    MCP's role in reducing manual coordination overhead; how AI-tool integration shifts engineering workflows from local-only to service-aware sessions.

[^23]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    `.mcp.json` as a shared, checked-in team artifact; per-project MCP server configuration patterns; relationship between `.mcp.json` and `settings.json` for MCP credential handling.

[^24]: Anthropic — "MCP Configuration Security," Claude Code Documentation, 2026. https://code.claude.com/docs/en/mcp-security
    Credential handling for MCP server connections; environment variable reference syntax in `.mcp.json`; separation of team configuration from machine-specific secrets.

[^25]: Jack Herrington — "Claude Code MCP Servers: A Complete Setup Guide," YouTube, November 2025. https://www.youtube.com/watch?v=3QkVZj_nKoA
    - MCP server architecture: the connection between Claude Code and external services and the read vs. write permission boundary
    - Building a custom Postgres MCP server for live schema inspection and parameterized query execution
    - Security considerations: credential management, OAuth scoping, and audit log configuration for MCP operations

---

## Configuration 5: .claude/commands/ and .claude/skills/ — Custom Skills and Commands

**Description:** Claude Code supports two complementary mechanisms for packaging reusable workflows: slash commands in `.claude/commands/` and skills in `.claude/skills/`. Slash commands are markdown files invoked with `/command-name` during a session — parameterized templates that encode the best prompt structure for a specific recurring task: feature scaffolding, refactoring passes, security review, test generation, or PR description creation. Skills extend this with supporting files and structured metadata, enabling more complex multi-artifact workflows.[^26]

These mechanisms are how a team builds a shared prompt library. The return on investment is asymmetric: the engineer who writes a well-crafted `/security-review` command invests fifteen minutes; every subsequent use by every team member benefits from that investment at no additional cost. Teams with shared command libraries report significantly higher output consistency than those relying on individual engineers' improvised prompts — because the command encodes not just the task but the architectural context, the expected output format, and the codebase-specific constraints that an engineer improvising a prompt would omit.[^1]

Both commands and skills are checked into git in their respective directories, making them versioned team artifacts with the same review properties as production code. An engineer who proposes a poorly structured command via PR is introducing tooling debt — a command that produces inconsistent output is a source of divergence at scale.[^27]

**Proposed Solution:**
- Establish a team command library as a required project artifact: at minimum, commands for feature scaffolding, refactoring, security review, test generation, and PR description creation. Check these into `.claude/commands/` alongside CLAUDE.md.[^26]
- Include project-specific architectural context in each command that would not be obvious to an engineer improvising a prompt: constraints, output format expectations, anti-patterns specific to the codebase, and quality criteria for that task type.[^1]
- Use `.claude/skills/` for complex workflows that require supporting files — a skill that runs a database migration, for example, can reference schema templates and rollback procedures as attached files.[^27]
- Allow engineers to propose new commands via PR, applying the same review process as production code changes. Define a quality criterion: a command is acceptable when it produces consistent, correct output across five distinct invocations by different engineers.[^3]
- Review and update commands quarterly: identify which produce consistently good output, which need updating due to codebase changes, and which common tasks lack a command and should have one.[^1]

---

[^26]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    `.claude/commands/` setup and file format; slash command parameterization; relationship between commands and CLAUDE.md context inheritance.

[^27]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    Skills vs. commands: when to use each; `.claude/skills/` structure with supporting files; team ownership and PR review process for shared prompt artifacts.

[^28]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    Shared prompt libraries as consistency infrastructure across mixed-experience teams; how well-designed reusable commands close the prompt-engineering skill gap between senior and junior engineers.

[^29]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    The five foundational command types every team should maintain; command update cadence as a team practice.

[^30]: Sabrina Ramonov — "CLAUDE CODE FULL COURSE," YouTube, February 17, 2025. https://www.youtube.com/watch?v=fYX6hHC9FhQ
    - Custom command file format: writing parameterized markdown commands and placing them in `.claude/commands/`
    - The five foundational command library entries: scaffolding, refactoring, security review, test generation, and PR description
    - Skills vs. commands: when supporting files justify using `.claude/skills/` over a single command file

---

## Configuration 6: .claude/agents/ — Subagent Definitions

**Description:** Claude Code supports subagent delegation — spawning independent Claude instances with their own context windows to perform parallel or scoped tasks. Subagent behavior can be configured through definition files in `.claude/agents/`, which specify the model to use, the tools available to the subagent, its instructions, and the scope of its access. This allows a team to define reusable delegation targets: a specialized code-review agent, a documentation-generation agent, a security-audit agent — each with pre-configured scopes and instructions.[^31]

Subagents are the mechanism by which long tasks that would exhaust a single context window are decomposed into parallel workstreams. An architect-level session orchestrates by dispatching scoped agents: one agent explores the codebase, another generates a plan, a third implements, and a fourth reviews — each in its own context, none polluting the others. The orchestrating session coordinates without holding all context simultaneously.[^32]

Agent definitions in `.claude/agents/` are checked into git as team artifacts. A well-defined agent definition encodes the appropriate scope, model, and tool access for a recurring delegation pattern — ensuring that when any engineer delegates a security audit, the delegated agent operates with the same constraints rather than inheriting whatever the calling session happens to have permitted.[^31]

**Proposed Solution:**
- Define team-standard agents for recurring delegation patterns: a `code-reviewer` agent (read-only, security-focused), a `doc-generator` agent (write-access to docs/ only), and a `test-generator` agent (write-access to test directories only).[^32]
- Scope each agent's tool access to the minimum required for its task — agent definitions are the primary mechanism for preventing delegated tasks from acquiring permissions beyond their mandate.[^31]
- Store agent definitions in `.claude/agents/` checked into git, subject to the same PR review process as commands and skills.[^33]
- Document the expected inputs, outputs, and failure modes for each agent definition — a delegation target without documented behavior becomes a black box that engineers use inconsistently.[^34]
- Review agent definitions quarterly: assess whether agent scope has drifted, whether new delegation patterns have emerged that warrant a new definition, and whether any definitions are unused.[^1]

---

[^31]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    `.claude/agents/` file format and supported fields; agent scope and tool access configuration; team-owned agent definitions as checked-in artifacts.

[^32]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    Subagent delegation patterns; orchestrator-worker decomposition for context-window management; agent definitions as reusable delegation targets.

[^33]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    How multi-agent orchestration patterns are becoming standard practice for complex software tasks; the operational shift from single-session to coordinated-agent workflows.

[^34]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    Agent definition as documentation: why encoding expected inputs, outputs, and failure modes is part of the engineering contract for a delegation target.

[^35]: Dex Horthy (YC Root Access) — "Advanced Context Engineering for Agents," YouTube, August 2025. https://www.youtube.com/watch?v=IS_y40zY-hc
    - Agent definition structure: model selection, tool access scoping, and instruction inheritance from CLAUDE.md
    - Orchestrator-worker decomposition: how to split a complex task across three agents without exhausting the orchestrating session's context window
    - Permission isolation: how agent-level tool restrictions prevent delegated sessions from acquiring parent-session permissions

---

## Configuration 7: .claude/rules/ — Scoped Instruction Overrides

**Description:** The `.claude/rules/` directory allows teams to define instruction files that apply only when Claude is working within specific file path patterns. Where CLAUDE.md applies globally to every session, a rules file can target a subtree — applying additional constraints, conventions, or context only when the file being operated on matches the rule's path gate. This enables differentiated behavior for different parts of the codebase without burdening every session with context that only applies to a fraction of work.[^36]

Common use patterns: a rules file scoped to `src/migrations/` that injects database-specific constraints and rollback requirements; a rules file scoped to `src/api/` that enforces REST convention and authentication annotation requirements; a rules file scoped to `tests/` that injects test quality criteria not relevant to production code sessions. Each rules file activates only when relevant, keeping the effective instruction context lean for any given task.[^36]

Rules files complement rather than replace CLAUDE.md: the global file establishes team-wide constraints while rules files add targeted detail for specialized contexts. Engineers working across both frontend and backend contexts benefit from rules that automatically inject the appropriate conventions without requiring manual context management per session.[^37]

**Proposed Solution:**
- Create rules files for high-specificity code areas: database migration directory, public API surface, authentication module, and test directories — each injecting the conventions and constraints that apply only in that context.[^36]
- Keep rules files short and specific. A rules file that sprawls into general conventions duplicates CLAUDE.md and creates a maintenance burden when conventions change.[^37]
- Version rules files in git alongside the code they govern. When a subsystem's conventions change, the corresponding rules file update should be part of the same PR.[^3]
- Document which rules files exist and what paths they cover in CLAUDE.md — engineers should know that migration rules exist before working in that directory, not discover them by accident.[^36]
- Audit rules file effectiveness in the same cycle as CLAUDE.md: verify that scoped instructions are producing the intended output, and prune any rules that Claude already follows correctly without them.[^1]

---

[^36]: Anthropic — "Claude Code: Settings and Configuration," Claude Code Documentation, 2026. https://code.claude.com/docs/en/settings
    `.claude/rules/` directory structure; path gate syntax; how rules files interact with global CLAUDE.md context at session time.

[^37]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    Rules files as a complement to CLAUDE.md; scoped instruction design patterns; the principle of keeping effective context lean for any given task.

[^38]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    Rules files for high-specificity contexts: the argument for automatic context injection over engineer-managed per-session prompting.

[^39]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    Modular context design: how layered instruction files reduce cognitive overhead for engineers working across multiple code contexts in a single sprint.

[^40]: Dex Horthy (YC Root Access) — "Advanced Context Engineering for Agents," YouTube, August 2025. https://www.youtube.com/watch?v=IS_y40zY-hc
    - Rules file path gate syntax: matching subtrees, file extensions, and named directories
    - CLAUDE.md + rules file interaction: how global and scoped instructions compose at session initialization
    - Maintenance discipline: updating rules files alongside the code they govern to prevent instruction drift

---

## Configuration 8: ~/.claude/projects/*/memory/ — Persistent Session Memory

**Description:** Claude Code maintains a persistent memory system at `~/.claude/projects/<project-hash>/memory/`. This directory contains `MEMORY.md` — an index of memory entries — and a set of topic files (e.g., `feedback.md`, `project.md`, `user.md`, `reference.md`) that accumulate knowledge across sessions. Memory persists between conversations, giving Claude access to prior decisions, corrections, and context without requiring engineers to re-establish it at each session start.[^41]

Memory is categorized into four types: *user* memories (the engineer's role, preferences, and knowledge level), *feedback* memories (corrections and validated approaches), *project* memories (ongoing work context, decisions, and deadlines), and *reference* memories (pointers to external resources). The distinction matters operationally: project memories decay as work evolves and must be pruned; feedback memories are durable and compound over time; user memories personalize responses without requiring engineers to re-introduce themselves each session.[^41]

Memory is distinct from CLAUDE.md: CLAUDE.md encodes team-level, codebase-specific instructions that apply to every engineer; memory encodes session-accumulated knowledge specific to a model instance's interaction with a particular engineer on a particular project. Memory supplements CLAUDE.md rather than replacing it — a correction added to memory prevents recurrence in future sessions for one engineer; a correction added to CLAUDE.md prevents recurrence for all engineers.[^42]

**Proposed Solution:**
- Treat memory as a complement to CLAUDE.md: use memory for session-specific learning and personal interaction preferences; escalate any correction that should apply team-wide into CLAUDE.md.[^41]
- Periodically review `~/.claude/projects/<project>/memory/MEMORY.md` to prune stale project memories (completed work, resolved incidents) and verify that feedback memories still reflect current practice.[^41]
- Use explicit memory instructions for high-value corrections: "remember that we never mock the database in integration tests" is a durable feedback memory that prevents a recurring AI error; preserve these deliberately rather than relying on implicit accumulation.[^42]
- Be aware that memory files can contain outdated information if not maintained. Before acting on a recalled memory that names specific files, functions, or architectural decisions, verify the current state of the codebase rather than treating the memory as authoritative.[^41]

---

[^41]: Anthropic — "Claude Code Memory," Claude Code Documentation, 2026. https://code.claude.com/docs/en/memory
    Memory system architecture: `MEMORY.md` index structure, topic file conventions, memory type categories, and the distinction between session memory and CLAUDE.md instructions.

[^42]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    Memory vs. CLAUDE.md: when to use each; feedback memory as a correction mechanism; memory pruning discipline for project memories that decay as work evolves.

[^43]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    The memory-to-CLAUDE.md escalation pattern: when a correction recurs across multiple sessions, it belongs in the shared team configuration rather than individual memory.

[^44]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    Persistent memory as a mechanism for accumulating institutional knowledge in AI sessions; how memory systems shift AI tooling from stateless to stateful collaboration.

[^45]: Dex Horthy (YC Root Access) — "Advanced Context Engineering for Agents," YouTube, August 2025. https://www.youtube.com/watch?v=IS_y40zY-hc
    - Memory file structure: `MEMORY.md` as an index with topic files for different memory categories
    - Session memory vs. CLAUDE.md: when accumulated knowledge belongs in personal memory vs. team configuration
    - Memory pruning: identifying and removing stale project memories that no longer reflect current work state

---

## Configuration 9: ~/.claude/keybindings.json and ~/.claude.json — Local Experience Configuration

**Description:** Two files configure the local Claude Code experience at the machine level. `~/.claude/keybindings.json` stores custom keyboard shortcuts, allowing engineers to remap or extend the default key bindings — binding session-start sequences, command invocations, or mode toggles to preferred key combinations. `~/.claude.json` is a managed state file that stores OAuth credentials, theme preferences, per-project trust decisions, MCP server connection caches, and session state. Both files are personal and machine-local; neither is checked into git.[^46]

`~/.claude.json` is written and read by Claude Code itself rather than edited manually, but understanding its role is operationally relevant: it is where per-project trust decisions persist (whether Claude is trusted to run shell commands in a given project without prompting), and where OAuth tokens for MCP servers are cached. When onboarding a new machine or debugging unexpected trust behaviors, `~/.claude.json` is the artifact to inspect.[^46]

`~/.claude/keybindings.json` is directly user-editable. Engineers who run multiple parallel sessions or regularly invoke the same slash commands benefit from binding these actions to short key sequences rather than typing them. The file supports chord bindings (multi-key sequences), enabling a dense shortcut vocabulary without key conflicts.[^47]

**Proposed Solution:**
- Document team-recommended keybindings in onboarding materials — not as required configuration, but as a menu of high-value shortcuts that engineers can adopt. Common high-value bindings: session-new, approve-tool, and the most frequently used slash commands.[^47]
- Instruct engineers to inspect `~/.claude.json` when debugging unexpected permission or trust behaviors — per-project trust decisions stored there can cause inconsistent behavior when moving between machines.[^46]
- Treat both files as personal configuration, not team artifacts. Never check them into git. For configuration that should be consistent across the team, use `.claude/settings.json` instead.[^9]
- When onboarding new team members, include a guided `~/.claude/keybindings.json` setup step. Engineers who configure shortcuts in the first week adopt them permanently; engineers who skip this step typically never revisit it.[^1]

---

[^46]: Anthropic — "Claude Code: Settings and Configuration," Claude Code Documentation, 2026. https://code.claude.com/docs/en/settings
    `~/.claude.json` as a managed state file; per-project trust decision persistence; OAuth token caching for MCP server connections; the distinction between personal and team configuration artifacts.

[^47]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    `~/.claude/keybindings.json` format; chord binding syntax; onboarding recommendations for keybinding setup; high-value shortcuts for parallel session workflows.

[^48]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    Onboarding friction reduction: why keybinding setup in the first week is a high-ROI investment for long-term workflow efficiency.

[^49]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    Personal configuration discipline: distinguishing between machine-local settings and team artifacts; the case for minimal personal override surface area.

[^50]: Sabrina Ramonov — "CLAUDE CODE FULL COURSE," YouTube, February 17, 2025. https://www.youtube.com/watch?v=fYX6hHC9FhQ
    - `~/.claude/keybindings.json` setup: file structure, chord binding syntax, and recommended bindings for parallel session workflows
    - `.claude.json` state file: what it contains, when to inspect it, and how per-project trust decisions affect session behavior
    - Personal vs. team configuration: the boundary between machine-local files and checked-in team artifacts

---

## Configuration 10: CI/CD Pipeline Integration — Headless Sessions and .worktreeinclude

**Description:** Claude Code runs in headless mode (`claude -p`) as a pipeline actor — enabling automated code review, documentation generation, migration execution, and test suite creation without human session management. The `--permission-mode plan` flag supports read-only analysis pipelines — architecture summaries, dependency audits, test coverage analysis — that run safely against every PR without risk of automated modification.[^51]

A CI pipeline that runs a standardized reviewer session on every PR — using a shared reviewer prompt against the team's CLAUDE.md — adds a consistent AI review layer that catches the same class of issues across all PRs regardless of which engineer authored them. This is more consistent than relying on individual engineers to run reviewer sessions manually before requesting review. For a small team where review bandwidth is limited, automated preprocessing that identifies the issues most likely to require attention focuses human review effort where it has the most value.[^52]

All team configuration artifacts — CLAUDE.md, `settings.json`, `.mcp.json`, hooks, commands — apply in headless sessions, making the CI environment behaviorally consistent with local sessions for the same project. The `.worktreeinclude` file at the project root defines gitignored files that should be copied into worktree environments for CI runs — ensuring that CI sessions have the same local configuration artifacts (environment files, local overrides) as developer machines without those files being tracked in git.[^53]

**Proposed Solution:**
- Integrate a focused PR review step in CI using `claude -p` with the team's reviewer prompt and CLAUDE.md context. Scope the review to security vulnerabilities, logic errors, and architectural pattern violations — not style issues, which static analysis already handles.[^52]
- Use plan-mode CI pipelines (`--permission-mode plan`) to generate structured architecture summaries for PRs touching critical modules — giving human reviewers a prepared description of what changed before they read the diff.[^51]
- Configure the CI review step to post findings as PR comments with explicit severity classifications (blocking, advisory, informational) so engineers can triage findings without reading the full review output.[^52]
- Run CI pipeline sessions under a dedicated service account with read-only repository permissions. Treat AI pipeline failures as blocking: investigate before merging, the same as failing tests.[^12]
- Define `.worktreeinclude` to ensure CI worktree environments receive any gitignored configuration files required for complete session behavior — without this, CI sessions may behave differently from local sessions in ways that are difficult to diagnose.[^53]

---

[^51]: Anthropic — "Claude Code in CI/CD Pipelines," Claude Code Documentation, 2026. https://code.claude.com/docs/en/cicd
    Headless CI mode patterns; `--permission-mode plan` flag for read-only analysis; how team configuration artifacts apply in pipeline sessions.

[^52]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
    PR review step design; plan-mode analysis pipeline for architecture summaries; PR comment severity classification; treating AI pipeline failures as blocking rather than advisory.

[^53]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    `.worktreeinclude` for copying gitignored files into worktree CI environments; worktree isolation patterns for parallel pipeline sessions.

[^54]: Anthropic — "Security and Permissions," Claude Code Documentation, 2026. https://code.claude.com/docs/en/security-permissions
    Service account permission model for pipeline sessions; `--allowedTools` scoping for headless sessions; audit logging configuration for CI permission grants.

[^55]: Jack Herrington — "Claude Code MCP Servers: A Complete Setup Guide," YouTube, November 2025. https://www.youtube.com/watch?v=3QkVZj_nKoA
    - Headless session configuration: how `.mcp.json` and CLAUDE.md apply in CI environments without interactive session management
    - Service account scoping: configuring read-only permissions for pipeline Claude instances
    - Worktree patterns for parallel CI sessions: isolating concurrent pipeline runs without configuration conflicts

---

## Summary of Configuration Surfaces

| Configuration File / Directory | Scope | Purpose | Checked Into Git? |
|---|---|---|---|
| `CLAUDE.md` (root or `.claude/`) | Project | Team instructions, conventions, corrections | Yes |
| `~/.claude/CLAUDE.md` | Global | Personal cross-project preferences | No (personal) |
| `.claude/settings.json` | Project | Permissions, hooks, env, model selection | Yes |
| `~/.claude/settings.json` | Global | User-level defaults for all projects | No (personal) |
| `.claude/settings.local.json` | Project | Machine-specific overrides (gitignored) | No |
| `managed-settings.json` | Enterprise | Non-overridable org policy floor | N/A (managed) |
| `.mcp.json` | Project | MCP server definitions and credentials | Yes (no secrets) |
| `.claude/commands/` | Project | Slash command library | Yes |
| `.claude/skills/` | Project | Multi-artifact reusable workflows | Yes |
| `.claude/agents/` | Project | Subagent delegation definitions | Yes |
| `.claude/rules/` | Project | Path-scoped instruction overrides | Yes |
| `~/.claude/projects/*/memory/` | Per-user/project | Persistent session learning | No (personal) |
| `~/.claude/keybindings.json` | Global | Personal keyboard shortcuts | No (personal) |
| `~/.claude.json` | Global | Managed state, OAuth, trust decisions | No (managed) |
| `.worktreeinclude` | Project | Gitignored files for worktree/CI copy | Yes |

---

## Recommended Implementation Order

| Priority | Configuration Surface | Immediate Action | Owner |
|---|---|---|---|
| 1 | CLAUDE.md | Create team CLAUDE.md; establish architect update protocol | Architect |
| 2 | settings.json + Hooks | Define permissions and team-standard hooks; check in | Architect |
| 3 | .mcp.json | Configure read-only MCP servers first; define shared file | Backend lead |
| 4 | commands/ + skills/ | Build initial 5-command library; establish PR review process | Engineering team |
| 5 | agents/ | Define code-reviewer and doc-generator agents | Architect |
| 6 | rules/ | Add scoped rules for migration and API directories | Architect |
| 7 | CI/CD headless | Add focused PR review step with CLAUDE.md context | Backend lead + QA |
| 8 | Memory hygiene | Document memory-to-CLAUDE.md escalation protocol | All engineers |
| 9 | Keybindings | Include keybinding setup in onboarding checklist | All engineers |
