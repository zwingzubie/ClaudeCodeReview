## Hooks and Automation: Enforcing Standards at the Session Layer

**Related to:** [Tooling Overview](00-overview.md) — Tool 2 · [Tooling: CLAUDE.md Configuration](01-claude-md-configuration.md)[^a] · [Governance: Review Policies](../Governance/01-review-policies.md)[^b] · [QA & Testing: AI-Generated Test Coverage](../QA%20%26%20Testing/02-ai-generated-test-coverage.md)[^c] · [Security: SAST and DAST Integration](../Security/02-sast-dast-integration.md)[^d]

---

## Overview

Claude Code's hook system is the mechanism by which team standards become enforced behaviors rather than aspirational guidelines. Without hooks, quality gates depend entirely on individual engineer discipline: running linting after a session, triggering tests before marking a task complete, scanning for security vulnerabilities before committing. With hooks, these operations happen automatically at defined event points in every session regardless of which engineer is running it and regardless of whether they remembered to do it manually. The gap between "we expect engineers to do X" and "engineers always do X" is the gap that hooks close.[^1]

This memo covers the hook event model, the four most valuable hook configurations for a small team (quality gates, context injection, notification, and security scanning), how to standardize hooks across all developer machines via project settings files, and the audit practices that verify hooks are functioning as intended. The technical ceiling of what hooks can do is high — this memo focuses on the subset that delivers the most governance value with the least configuration overhead.

---

## Section 1: The Hook Event Model

**Description:** Claude Code fires hooks at six event types in a session lifecycle: `PreToolUse` (before Claude uses any tool), `PostToolUse` (after a tool call completes), `UserPromptSubmit` (when the engineer submits a new prompt), `Stop` (when Claude finishes a response), `Notification` (when Claude sends a system notification), and `SubagentStop` (when a subagent session completes). Each hook type receives structured JSON input about the event — the tool name, the file being written, the command being run — and can return output that Claude incorporates into its next action.[^2]

The most important distinction for governance purposes is between blocking and non-blocking hooks. A hook that exits with a non-zero status code blocks the Claude action that triggered it — the file write does not complete, the command does not run. A hook that exits with zero status allows the action to proceed regardless of what the hook output says. Most quality gate hooks should be configured as blocking: a linting failure should prevent the file write from completing, not produce a warning that Claude proceeds through.[^2]

**Recommended Practice:**
- Configure quality gate hooks as blocking (non-zero exit on failure) and informational hooks as non-blocking (always exit zero). The former prevent incorrect actions; the latter add information without adding friction to correct ones.[^2]
- Keep hook scripts short (under 20 lines each) and focused on a single responsibility. Complex multi-step hook scripts become maintenance burdens; separate concerns into separate hooks that each do one thing well.[^3]
- Test hook behavior explicitly in a dev environment before deploying to the team: trigger the events that should fire each hook and verify that blocking hooks actually block, non-blocking hooks log correctly, and the JSON input structure matches what the hook expects.[^1]
- Document each hook in the `.claude/hooks/` README with: what event it fires on, what it does, whether it is blocking, and how to override it for legitimate exceptions. Engineers who encounter unexpected blocking need to understand why and how to proceed.[^2]

---

## Section 2: Quality Gate Hooks

**Description:** Quality gate hooks are PostToolUse hooks that fire after Claude writes files, runs commands, or modifies the codebase. They are the automated enforcement layer for standards that would otherwise rely on individual engineers to apply manually before finishing a session. The three most valuable quality gate hooks for a small team are: a lint hook that runs after every file write (catching style violations before they reach review), a test hook that runs after source file changes (verifying that changes do not break existing tests), and a build hook that runs after significant changes (ensuring the project still compiles before the session ends).[^4]

The specific value of hooking these operations to Claude's tool use, rather than just running them at CI time, is that Claude can see the hook output and respond to it within the same session. A linting failure surfaced in a PostToolUse hook produces an error that Claude reads, interprets, and acts on — fixing the lint issue without engineer intervention. The same failure surfaced in CI produces a failing check that requires the engineer to re-open the session, diagnose the issue, and make the correction manually. Hooks move the feedback loop from hours (CI time + human review) to seconds (within the current session).[^4]

**Recommended Practice:**
- Configure a PostToolUse hook on `Write` events for all source file types: run the project linter with a non-zero exit on errors. File the hook in `.claude/settings.json` as a project-level setting checked into git so all team members share the same configuration.[^2]
- Configure a PostToolUse hook on `Write` events for test-adjacent source files: run the relevant test suite (or just the tests for the modified module) with a non-zero exit on failures. Keep this scoped — running the full test suite on every file write will slow sessions to a crawl; run only the tests relevant to changed files.[^3]
- For security-sensitive directories (`src/api/`, `src/auth/`, database migration directories), configure a specialized PostToolUse hook that runs SAST scanning on modified files. This adds a targeted security gate in exactly the places where AI-generated code poses the highest risk.[^5]
- Add a Stop hook that runs the full test suite and generates a brief session completion summary. This catches any accumulated failures from a long session, giving the engineer a clear picture of session state before they commit and sign off.[^4]

---

## Section 3: Context Injection Hooks

**Description:** The UserPromptSubmit hook fires whenever the engineer submits a prompt at the beginning of a response cycle. Unlike CLAUDE.md (which is read once at session start), UserPromptSubmit hooks fire repeatedly — on every prompt — and can inject dynamic context that is not available at session start. This enables automatic injection of date-stamped context, current sprint information, recent git log summaries, or other information that varies per session and would otherwise require the engineer to supply it manually.[^6]

Context injection hooks are most valuable for information that is always relevant but easily forgotten: the current date (critical for avoiding AI suggestions based on outdated practices), recent commits in the affected files (providing awareness of concurrent changes), and any team alerts or architectural notes that were added since the session began. Without these hooks, this context either requires manual engineer effort to provide or goes missing — and a session that doesn't know the current date may suggest approaches that were superseded months ago.[^2]

**Recommended Practice:**
- Configure a UserPromptSubmit hook that injects the current date and recent git log (last three commits for the files in scope) into every prompt. This is a four-line shell script that pays disproportionate returns on sessions involving rapidly evolving files.[^6]
- For teams using a sprint planning system (Linear, Jira), add a hook that fetches and injects the current sprint's goal and active tickets in scope. This connects Claude's session-level awareness to the team's actual in-flight priorities rather than requiring engineers to describe context they have already entered elsewhere.[^3]
- Use the PreToolUse hook on Bash execution to log every command Claude runs to an audit file. This is not a blocking hook — it does not prevent execution — but it creates a session-level audit trail that can be reviewed after the fact if an unexpected change occurred during an unattended session.[^2]
- Validate context injection hooks by starting a session and reviewing the first prompt Claude receives before it generates a response. If the hook is working, the injected context should appear in Claude's input. If not, the hook script has a bug or the injection format is incorrect.[^1]

---

## Section 4: Notification and Coordination Hooks

**Description:** Parallel sessions — running multiple Claude instances simultaneously on separate git worktrees — are one of the highest-leverage productivity patterns in Claude Code. Boris Cherny described running five instances simultaneously as his primary productivity multiplier.[^7] But parallel sessions require coordination: engineers need to know when a session completes its task so they can review the output, decide on next steps, and launch the next session. Without notification hooks, coordination requires constant active monitoring — checking back on sessions manually, waiting around for long operations to finish.[^8]

The Notification hook fires when Claude sends a system notification (typically when a long task completes, when input is needed, or when an error requires human attention). Configuring this hook to send a push notification, a Slack message, or a system alert transforms parallel session coordination from active monitoring to passive notification — engineers can work on other tasks while sessions run and receive an interrupt only when attention is needed.[^2]

**Recommended Practice:**
- Configure a Notification hook that sends a system-level alert (macOS notification, desktop popup) when Claude completes a long task or needs input. The hook configuration is a single shell command: `osascript -e 'display notification "Claude session needs attention" with title "Claude Code"'` on macOS.[^2]
- For teams using Slack, configure the Notification hook to post a brief message to an engineering channel when a session completes a significant milestone. This enables the team to be aware of parallel sessions without requiring a dedicated monitoring workflow.[^3]
- Add a SubagentStop hook that logs the subagent's task completion and any findings to a shared session log file. When multiple engineers are running parallel sessions, a shared log enables coordination without requiring synchronous communication.[^6]
- Configure the SubagentStop hook to run a lightweight output audit before the orchestrator session can accept a subagent's changes: verify that modified files pass the lint hook, that any new functions have corresponding test stubs, and that the subagent's reported change list matches the actual file diff (using `git diff --name-only` against the pre-session state). A mismatch between reported and actual changes is the most common agentic session failure mode — the subagent modified files it was not tasked with, or left a file in a partially modified state. The hook should block orchestrator commit until the diff matches the spec's file scope, exiting non-zero on mismatch so the orchestrator session halts rather than proceeding silently.[^3][^8]
- Use the Stop hook to automatically generate a brief session summary (task completed, files modified, tests passing/failing) and write it to a timestamped file in a `.claude/sessions/` directory. These summaries are valuable for sprint retrospectives and for resuming sessions across working days.[^8]

---

## Section 5: Standardizing Hooks Across the Team

**Description:** A hook configuration that exists only on one engineer's machine provides no team-level governance. The value of hooks is proportional to their consistency: a team where every engineer has the same quality gate hooks enforces the same standards in every session; a team where hooks are individually configured enforces inconsistent standards and produces inconsistent output. Team-level hook standardization requires a configuration artifact that is version-controlled, distributed as part of the repository, and applied automatically when engineers set up their development environment.[^1]

Claude Code supports project-level hook configuration via `.claude/settings.json` checked into git. When this file is present, Claude Code loads it automatically for all sessions in that repository — meaning hooks defined there apply to every engineer who checks out the repository without any individual setup step. Project-level configuration does not prevent engineers from having personal overrides in their user-level settings, but it ensures the team standard is always present as the baseline.[^2]

**Recommended Practice:**
- Define all team-standard hooks in `.claude/settings.json` in the repository root. Check this file into git with the same review requirements as CLAUDE.md — it is load-bearing configuration that affects all engineers' sessions.[^2]
- Document the rationale for each hook configuration in the `.claude/settings.json` file using comments or an accompanying README. Engineers who encounter an unfamiliar hook blocking their session need to understand why before they can evaluate whether to override it.[^1]
- Run a quarterly hook audit: verify that all team-standard hooks are present and functioning correctly in a clean session, confirm that no engineer has personal overrides that bypass critical quality gates, and review whether new hook types should be added based on recurring session issues identified in the monthly AI practice review.[^3]
- When onboarding new engineers, include a hook walkthrough as part of the Claude Code setup session: explain what each hook does, when it fires, and how to handle the most common blocking scenarios. Engineers who understand the hooks are less likely to bypass them; engineers who encounter unexplained blocks are more likely to disable them.[^4]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Hook Event Model | Document blocking vs. non-blocking policy for each hook type | Architect |
| Quality Gate Hooks | Configure lint + test PostToolUse hooks in .claude/settings.json | Architect + Backend lead |
| Context Injection Hooks | Add date + recent git log UserPromptSubmit hook | Architect |
| Notification Hooks | Configure system notification Stop hook for parallel session coordination | All engineers |
| Team Standardization | Verify .claude/settings.json is committed and applied on all developer machines | Architect |

---

[^1]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    Team-level hook standardization as governance infrastructure: the difference between aspirational standards and enforced standards in AI-assisted development.

[^2]: Anthropic — "Hooks Reference," Claude Code Documentation, 2026. https://code.claude.com/docs/en/hooks-reference
    Complete hooks API: event types, JSON input structure, blocking semantics, project-level settings.json configuration, and hook ordering for multi-hook event handlers.

[^3]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    Hook use cases: quality gate patterns, context injection examples, parallel session coordination with notification hooks, and the SubagentStop hook for multi-agent coordination.

[^4]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
    Hooks as session-layer quality gates: the argument for moving feedback loops from CI time to session time; how blocking hooks change Claude's response to failures vs. advisory hooks.

[^5]: Veracode — "Spring 2026 GenAI Code Security Update: Despite Claims, AI Models Are Still Failing Security," March 24, 2026. https://www.veracode.com/blog/spring-2026-genai-code-security/
    Security scanning at the session layer: why targeted SAST hooks on security-sensitive directories address the 45% AI code security failure rate more effectively than CI-only scanning.

[^6]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    Context injection hooks and dynamic context: the UserPromptSubmit hook for injecting session-specific context that CLAUDE.md cannot provide; parallel session coordination via notification hooks.

[^7]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
    Parallel session workflow: running five sessions simultaneously as the primary productivity unlock; notification hooks as the coordination mechanism that makes this workflow practical.

[^8]: Anthropic — "2026 Agentic Coding Trends Report," Anthropic, 2026. https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf
    Agentic session coordination: how notification and Stop hooks enable engineers to manage unattended sessions without constant monitoring; the session summary pattern for long-running delegated tasks.

[^9]: DEV Community — "AI Is Creating a New Kind of Tech Debt — And Nobody Is Talking About It," March 2026. https://dev.to/harsh2644/ai-is-creating-a-new-kind-of-tech-debt-and-nobody-is-talking-about-it-3pm6
    Hooks as the enforcement layer for governance policies: the argument that team standards embedded in automated hooks are more reliable than standards dependent on individual memory and discipline.

[^10]: Sonar (SonarSource) — "Sonar Data Reveals Critical 'Verification Gap' in AI Coding," press release, January 8, 2026. https://www.sonarsource.com/company/press-releases/sonar-data-reveals-critical-verification-gap-in-ai-coding/
    The verification gap — 52% of developers who distrust AI code accept it without verification — as the specific behavioral failure that quality gate hooks address by making verification automatic rather than voluntary.

[^11]: Sabrina Ramonov — "The ULTIMATE Claude Code Tutorial," YouTube, February 17, 2026. https://www.youtube.com/watch?v=fYX6hHC9FhQ
    - Step 3 (Quality Gate): complete walkthrough of configuring PostToolUse hooks for linting, testing, and security scanning with blocking behavior for failures
    - UserPromptSubmit hook configuration: live demonstration of dynamic context injection including date, sprint context, and recent git activity
    - settings.json team configuration: how to structure the project-level settings file for consistent hook distribution across all developer machines

[^12]: NetworkChuck — "I Automated My Entire Code Quality Pipeline With Claude Hooks," YouTube, February 2026. https://www.youtube.com/watch?v=5aQvKhLb2mE
    - Building a complete hook pipeline: lint → test → SAST → notify in a sequential PostToolUse chain for source file writes
    - Notification hook configuration: practical setup for macOS system notifications and Slack integration for parallel session coordination
    - Audit logging: PreToolUse hook on Bash commands that creates a per-session command log for security review and debugging

[^13]: Dex Horthy (YC Root Access) — "Advanced Context Engineering for Agents," YouTube, August 2025. https://www.youtube.com/watch?v=IS_y40zY-hc
    - UserPromptSubmit hooks for dynamic context: how automated context injection removes the engineer burden of supplying standard context manually in every session
    - Hook composition: how to build multi-stage hook pipelines that gate on sequential conditions rather than treating each hook as independent
    - Team standardization: how project-level settings files ensure hook consistency without requiring individual engineer setup steps

[^a]: [Tooling: CLAUDE.md Configuration](01-claude-md-configuration.md) — hooks extend CLAUDE.md constraints into executable enforcement; the two tools together form the complete session governance layer.

[^b]: [Governance: Review Policies](../Governance/01-review-policies.md) — hooks automate the pre-commit and pre-push checks that review policies require; automated enforcement reduces the reviewer burden and prevents policy drift.

[^c]: [QA & Testing: AI-Generated Test Coverage](../QA%20%26%20Testing/02-ai-generated-test-coverage.md) — coverage analysis hooks run test coverage automatically on AI-generated code; the hook triggers what the QA section defines as the coverage requirement.

[^d]: [Security: SAST and DAST Integration](../Security/02-sast-dast-integration.md) — SAST scanning can be integrated as a pre-commit hook; the two documents describe the security tool and its automation integration point.
