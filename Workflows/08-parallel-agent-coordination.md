## Parallel Agent Coordination

**Related to:** [Workflows Overview](00-overview.md) — Workflow 8

---

## Overview

Parallel agent coordination is the practice of running multiple Claude Code sessions simultaneously on independent but related tasks, and then integrating their outputs. It is the workflow pattern that converts Claude Code from a single-engineer acceleration tool into a team-scale force multiplier: instead of one session completing tasks sequentially, multiple sessions complete independent tasks in parallel, with total elapsed time determined by the longest task rather than the sum of all tasks.

The case for parallelism is straightforward when tasks are genuinely independent: a migration that affects forty files in four separate modules can be done in four parallel sessions, each scoped to one module, with outputs merged afterward. What had been a two-day sequential effort becomes a four-hour parallel effort followed by an integration review. But parallel coordination has failure modes that sequential work does not — shared state conflicts, session divergence, integration failures at merge time — and these failure modes scale with the number of sessions if the coordination architecture is not deliberate.[^1]

This memo covers when parallel agent workflows are appropriate and when they are not, the coordination patterns that prevent session divergence and shared state conflicts, the git worktree setup that makes parallel sessions safe, the synchronization checkpoints that prevent parallel sessions from diverging on shared dependencies, and the failure handling protocol when one or more sessions fail mid-run. The operational context throughout is a team of 11 where the backend engineers own parallel session workflows and the architect owns the coordination architecture decisions.[^2]

---

## Section 1: When Parallelism Is Appropriate

**Description:** Parallel agent coordination adds value when tasks are genuinely independent: they do not read or write the same files, they do not depend on shared state that any session might modify, and their outputs can be integrated through a merge process without requiring sequential reasoning about the combined result. These conditions are more restrictive than they appear. Many tasks that seem independent share a schema, a type definition, a shared utility, or an interface contract that all sessions will need to reference — and if any session modifies that shared element, the other sessions are operating against stale context.[^3]

The task types that consistently meet the independence criteria are large-scale repetitive migrations (applying the same transformation to N files in N independent modules), multi-module feature implementations where modules have clean interfaces defined in advance (backend API implementation in parallel with frontend consumer implementation, when the interface contract is frozen before either session starts), and test suite generation (writing tests for N independent modules simultaneously, when no module's tests depend on another module's implementation details).[^1]

The task types that do not meet the independence criteria, regardless of how they are framed: anything involving a shared database schema that is under active modification, anything involving shared TypeScript types or interfaces that have not been finalized, anything involving authentication or authorization logic that is referenced across multiple modules, and anything where the correct implementation of one module depends on understanding the implementation details of another. These are fundamentally sequential tasks dressed in parallel framing, and attempting to run them in parallel produces integration failures that cost more time to resolve than the parallelism saved.[^2]

**Recommended Practice:**
- Apply a two-question independence test before launching parallel sessions: (1) Can each session complete its task without reading any file that another session will write? (2) Can each session's output be integrated into a coherent codebase without requiring any session to know how another session implemented its task? A "no" to either question means the task set is not ready for parallelism.[^3]
- For large migrations, run a single-session pilot on one module before launching parallel sessions for the remaining modules. The pilot validates the prompt and approach; parallel sessions on top of a validated approach succeed at a much higher rate than parallel sessions on top of an untested approach.[^1]
- Document the interface contracts and shared type definitions that parallel sessions will rely on before any session starts. These documents are the coordination anchor — each session reads them at start, and no session is permitted to modify them. Freeze the contract before the sessions start; if the contract needs to change, abort the parallel run and update it first.[^4]
- For the team's typical sprint, identify which backlog items have genuine independence and which are sequential masquerading as parallel. This analysis should happen at sprint planning, not at session launch time. Parallel coordination that is planned in advance produces better task decomposition than parallelism decided ad hoc.[^2]

---

## Section 2: Coordination Patterns

**Description:** The core coordination challenge in parallel agent sessions is preventing the sessions from making incompatible assumptions about shared context. Each session has its own context window, its own understanding of the codebase at session start, and its own approach to the task. Without a coordination architecture, two sessions implementing adjacent modules will make different assumptions about how those modules should interact — assumptions that only reveal their incompatibility at merge time, after both sessions are complete.[^4]

Three coordination patterns address this. The shared spec document pattern gives each session a written specification that covers the interface between its module and every other module, the shared data models, the error handling contract, and the naming conventions that all sessions must follow. The spec is written before any session starts; each session reads the spec at session start and treats it as binding. The file scope assignment pattern assigns each session an explicit list of files it is authorized to create or modify and prohibits modification of any file outside that list. Violations of file scope produce merge conflicts and integration failures that are visible and traceable. The designated merge agent pattern assigns one session — typically the last one to run, after parallel outputs are ready — the task of integrating the parallel outputs and resolving any conflicts. The merge agent has context about what each parallel session did and why, making integration decisions more coherent than a human merge without that context.[^1]

These patterns are not mutually exclusive; the most robust parallel coordination uses all three. The shared spec prevents semantic conflicts; file scope assignment prevents file-level conflicts; the merge agent handles residual integration complexity. The overhead of all three is high relative to a single-session workflow, which is why parallelism should be reserved for tasks where the time savings are proportionally large.[^3]

**Recommended Practice:**
- Write the shared spec document as the first step of any parallel agent workflow. The spec must cover: module boundaries (what each session owns), shared interfaces (API contracts, data models, error types), naming conventions, and prohibited patterns. If writing the spec takes more than two hours, the tasks are probably not well-decomposed for parallel execution.[^4]
- Assign file scope explicitly in the session prompt: "You are responsible for files in `src/modules/auth/`. Do not create or modify any file outside this directory." Include the scope assignment in the session-start CLAUDE.md context so it is visible throughout the session, not just at session start.[^1]
- For complex parallel workflows, designate one engineer as the coordination owner who writes the spec, monitors parallel sessions for scope violations, and runs the merge agent session. Parallel workflows without a coordination owner are more likely to produce integration failures that require individual session authors to diagnose.[^2]
- After the first parallel workflow using a new task decomposition pattern, conduct a brief retrospective: did the sessions stay within their file scope? Did the shared spec cover all the interface decisions that came up? Were there integration failures that better spec writing would have prevented? Use the retrospective findings to improve the spec template for future parallel workflows of this type.[^3]

---

## Section 3: Git Worktree Setup for Parallel Sessions

**Description:** Parallel agent sessions require worktree isolation as a prerequisite. Two Claude Code sessions running against the same working directory will produce file conflicts that corrupt both sessions' output — the sessions share no coordination mechanism at the filesystem level and will overwrite each other's changes without warning. Git worktrees provide each session with a completely isolated working directory on its own branch, so session outputs are independent and can be reviewed and merged through the standard PR process.[^5]

The full worktree setup procedure for a parallel workflow is: create one worktree per session before any session starts, assign each session to its worktree in the session prompt, run sessions in parallel with each session operating exclusively in its assigned worktree, review each worktree's output as a PR, and merge through a designated integration branch. The worktree isolation means that each session's output is reviewable independently — a session that produced incorrect output can be discarded without affecting the other sessions' outputs.[^1]

For the team's practical workflow, the standard parallel session setup is:

```bash
git fetch origin
git worktree add ../repo-claude-auth -b claude/parallel/auth origin/main
git worktree add ../repo-claude-api -b claude/parallel/api origin/main
git worktree add ../repo-claude-frontend -b claude/parallel/frontend origin/main
```

Each session is then launched with its working directory set to the appropriate worktree path. After all sessions complete, each worktree branch is opened as a PR for review before any merging occurs. The full worktree lifecycle — creation, session, review, merge, removal — is documented in [Tooling/07-worktree-isolation.md](../Tooling/07-worktree-isolation.md).

**Recommended Practice:**
- Create all session worktrees from the same base commit before any session starts. If sessions start from different base commits — because main received a merge between the first and last worktree creation — the integration will be more complex than necessary. Create all worktrees atomically: `git fetch origin` once, then create all worktrees in immediate succession.[^5]
- Name parallel session worktrees with a shared prefix that identifies the parallel workflow: `claude/parallel/auth-refactor-auth`, `claude/parallel/auth-refactor-api`, `claude/parallel/auth-refactor-frontend`. The shared prefix makes the parallel relationship visible in the branch list and simplifies cleanup.[^1]
- For parallel workflows with more than three sessions, script the worktree creation to prevent naming errors. A script that reads a session list and creates worktrees with consistent names is more reliable than manual creation at scale.[^4]
- Remove all worktrees from a parallel workflow as a batch after all PRs are merged. Do not leave partial cleanup — a set of parallel worktrees where some have been removed and some have not creates confusion about which outputs have been integrated and which are pending.[^5]

---

## Section 4: Synchronization Points

**Description:** Parallel sessions that must proceed to a second phase after the first phase completes require synchronization points — explicit checkpoints where all sessions must complete their current phase before any session proceeds to the next phase. The most common synchronization requirement is a shared dependency that phase one may discover needs to change: a database schema that turns out to need an additional column, a TypeScript interface that turns out to need an additional field, a shared utility that turns out to need a new parameter. If any parallel session identifies this need and modifies the shared dependency while other sessions are still operating with the original, the result is diverged implementations that will fail integration.[^2]

The synchronization point procedure is: complete phase one across all sessions, review all phase-one outputs before any session begins phase two, identify any changes to shared interfaces or shared data models that phase one revealed, update the shared spec document with these changes, and then start phase two sessions from the updated spec. This procedure adds calendar time but eliminates integration failures that would require re-running phase two entirely — a much worse outcome than the planned delay.[^3]

Synchronization points are required whenever parallel sessions share any of: database schema definitions, TypeScript type or interface definitions used across module boundaries, shared utility function signatures, shared configuration structures, or API contracts between the modules being implemented in parallel. These are the elements whose modification by one session makes all other sessions' work inconsistent with the new state.[^1]

**Recommended Practice:**
- Identify all synchronization points before the parallel workflow starts, as part of the shared spec document. Document each synchronization point as: what shared element it protects, which sessions depend on it, and what the review process is before phase two begins.[^4]
- Configure each parallel session to flag modifications to shared elements rather than making them unilaterally. The session prompt should specify: "If you determine that [shared interface] needs to change to complete your task, do not make the change. Instead, stop and report what change is needed in your session output." This converts a silent shared-state modification into an explicit synchronization request.[^2]
- After phase one completes, run a synchronization review before phase two starts. The coordination owner reviews all phase-one outputs for flagged shared-element change requests and for unreported shared-element modifications (visible through the worktree diffs). The review should take no more than 30 minutes for a typical three-to-five session parallel workflow.[^3]
- Treat any unreported shared-element modification discovered during synchronization review as a session failure. The session exceeded its file scope assignment and its output must be revised before phase two proceeds. Document the violation in the post-workflow retrospective as feedback for improving scope assignment in future parallel workflows.[^1]

---

## Section 5: Failure Handling in Parallel Runs

**Description:** Parallel sessions fail for the same reasons individual sessions fail — insufficient context, ambiguous requirements, verification failures — but the failure of one session in a parallel run creates decisions that individual session failures do not: should the other sessions be aborted or allowed to continue? Can the failed session's output be salvaged, or must it be discarded entirely? If the failed session touched a shared element, have the other sessions already incorporated assumptions based on that shared element's original state?[^3]

The failure decision framework depends on whether the failing session's output is in the critical path for the remaining sessions. If the failing session owned a module that other sessions depend on for their second phase, those sessions cannot proceed until the failure is resolved — abortthe remaining sessions and address the failure before relaunching. If the failing session owned an independent module with no shared dependencies, the other sessions can continue to completion while the failing session is relaunched independently.[^1]

Partial parallel output — three of five sessions complete, two fail — is the most complex integration scenario. The completed sessions have produced reviewed, mergeable output. The failed sessions have produced partial output that may contain useful work alongside incorrect work. The coordination owner must make a triage decision: does the partial output from failed sessions contain enough correct work to review and salvage, or is a full relaunch faster? This decision is easier to make correctly when session output is scoped to independent modules, because the review surface is bounded.[^4]

**Recommended Practice:**
- Define the abort criterion before the parallel workflow starts: if a session fails and it owns a shared element that other sessions depend on, all sessions are paused and the failure is addressed before the remaining sessions continue. Document this criterion in the session prompt for all parallel sessions so each session knows the abort condition.[^3]
- When a session fails, do not relaunch it immediately. Review its partial output first: identify the point of failure, determine whether the partial output before the failure point is correct, and decide whether to start the relaunch from the beginning (preserving only the spec context) or from a checkpoint (using the correct partial output as context for the remainder).[^1]
- For parallel workflows where sessions are scripted with `claude -p` loops (as described in [Workflows/07-agentic-delegation.md](07-agentic-delegation.md) Section 3), configure the loop to write session output to individual log files per worktree. Failed sessions are identified by FAIL responses; the log files allow the coordination owner to inspect partial output without opening each worktree directory individually.[^2]
- After any parallel workflow where one or more sessions failed, conduct a brief retrospective focused on the failure: was the failure caused by an insufficient shared spec, an overly broad file scope assignment, a missing synchronization point, or a task that should not have been parallelized? The retrospective finding should update the parallel workflow checklist for the next run.[^4]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Parallelism Appropriateness | Define independence test; identify parallelizable task types at sprint planning | Architect |
| Coordination Patterns | Write shared spec template; enforce file scope assignment; designate merge agent | Architect |
| Worktree Setup | Script parallel worktree creation; adopt parallel naming convention | Backend lead |
| Synchronization Points | Document synchronization points in shared spec; configure sessions to flag shared-element changes | Architect |
| Failure Handling | Define abort criterion before workflow starts; log session output to per-worktree files | Backend lead |

---

[^1]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    Fan-out patterns for parallel Claude invocations; `--allowedTools` scoping for parallel sessions; `claude -p` loop patterns for unattended parallel execution; per-session output logging with `--output-format stream-json`.

[^2]: Anthropic — "2026 Agentic Coding Trends Report," Anthropic, 2026. https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf
    Multi-agent coordination as an emerging team practice; parallel delegation patterns; the infrastructure prerequisites for safe parallel execution; bounded autonomy requirements for multi-session workflows.

[^3]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    Parallel agent coordination patterns in 2026; shared state conflicts as the primary failure mode; the role of pre-session specification in preventing session divergence; integration failure triage methodology.

[^4]: incident.io — "How We're Shipping Faster with Claude Code and Git Worktrees," incident.io Blog, 2026. https://incident.io/blog/shipping-faster-with-claude-code-and-git-worktrees
    Practical parallel session coordination at team scale: shared spec discipline, per-session worktree assignment, the merge agent pattern, and the retrospective process that improved coordination quality over successive parallel workflows.

[^5]: Tooling — "Git Worktrees for Parallel Session Isolation," ClaudeCodeReview, 2026. Tooling/07-worktree-isolation.md
    Complete worktree lifecycle for parallel sessions: atomic creation from shared base, branch naming conventions, review-before-merge requirement, batch removal after integration. The foundational tooling prerequisite for all parallel agent coordination workflows.

[^6]: Tessl — "8 Agentic Coding Trends Shaping Software Engineering in 2026," March 2026. https://tessl.io/blog/8-trends-shaping-software-engineering-in-2026-according-to-anthropics-agentic-coding-report/
    Multi-agent specialization and parallelization as the dominant 2026 agentic trend; orchestrator/subagent patterns; synchronization point design for multi-phase parallel workflows; failure handling in multi-agent pipelines.
