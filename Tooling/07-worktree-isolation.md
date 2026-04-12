## Git Worktrees for Parallel Session Isolation

**Related to:** [Tooling Overview](00-overview.md) — Tool 7

---

## Overview

Parallel Claude Code sessions running against the same working tree are a collision waiting to happen. When two sessions both read the same files, plan modifications, and begin writing, the result is not two streams of parallel progress but a race condition: the second session to write a file overwrites the first session's changes, leaving the working tree in a state that reflects neither session's complete intent. This is not a theoretical risk — it is the practical consequence of running agentic sessions, which may modify dozens of files, against shared mutable state.

Git worktrees provide the isolation primitive that makes parallel agentic sessions safe. A worktree is a separate checked-out copy of the repository linked to the same git object store — each worktree has its own working directory, its own branch, and its own index, but shares history with the main clone. Two Claude Code sessions running in separate worktrees are operating on completely independent file states. Neither session can corrupt the other's working tree, because there is no shared working tree.

For a team working toward the parallel delegation patterns described in [Workflows/07-agentic-delegation.md](../Workflows/07-agentic-delegation.md) and [Workflows/08-parallel-agent-coordination.md](../Workflows/08-parallel-agent-coordination.md), worktrees are not optional infrastructure — they are the prerequisite that makes parallel sessions safe enough to use in a shared codebase. This memo covers the setup, naming conventions, lifecycle discipline, and governance that make worktree-based parallel sessions a sustainable team practice rather than an ad hoc experiment.

---

## Section 1: Why Worktree Isolation Matters for Parallel Agentic Work

**Description:** The problem with shared working tree parallelism is not just file-level conflict. It is also agent confusion. A Claude Code session that begins reading a set of files, forms a plan based on their contents, and then finds those files have changed mid-session has lost coherence between its plan and the actual state of the code. Depending on what the session does next, it may apply its planned edits to the new state (producing silent logical errors), fail on the changed files (producing a confusing error mid-session), or overwrite the changes entirely (losing the other session's work).[^1]

The working tree is the session's world model. For an agentic session — one that will proceed for minutes to hours without interactive steering — the world model must remain stable for the duration of the session. If the underlying files can change while the session is running, the session's output is non-deterministic in a way that is difficult to diagnose. The session cannot communicate about changes it did not observe, and the human reviewing its output has no visibility into whether the output was produced against the state the human intended.[^2]

Worktree isolation makes this problem disappear structurally. Each session operates in its own directory, on its own branch, with no interference from other sessions. When the session is complete, its output is reviewed as a coherent unit — a branch on a worktree that reflects exactly what that session did, starting from a known state, with no external modifications. Review becomes tractable because the output is bounded and attributable.[^3]

**Recommended Practice:**
- Treat worktree creation as a prerequisite for any parallel agentic session. Do not run two Claude Code sessions against the same working directory simultaneously — full stop. Document this constraint in CLAUDE.md as a team rule, not a suggestion.[^1]
- For single-session work, worktrees are optional. For multi-session work — fan-out migrations, parallel feature development, simultaneous backend/frontend implementation — worktrees are required. Make the distinction clear in the team's agentic delegation playbook.[^2]
- Educate the team on the failure mode before it is experienced in production. A brief walkthrough of how two sessions corrupt a shared working tree — and what the corrupted state looks like — produces more durable adoption than a policy document alone.[^3]
- Track incidents where working tree corruption occurred before worktree adoption was mandatory. Use these as training examples when onboarding new engineers to parallel session workflows.[^1]

---

## Section 2: Git Worktree Setup

**Description:** The `git worktree add` command creates a new worktree in a directory of the engineer's choosing, checked out to a new or existing branch. The setup is a single command, and the worktree is immediately available as a separate working directory. The only shared state between the main clone and the worktree is the git object store — the files themselves are independent copies, and changes in one do not appear in the other until they are committed and fetched/merged.[^4]

The standard setup for a Claude Code parallel session worktree is:

```bash
git worktree add ../my-repo-claude-session-name -b claude/session-name
```

This creates a sibling directory (adjacent to the main repo directory, not inside it) checked out to a new branch named `claude/session-name`. The sibling directory placement keeps the worktree out of the main repository's directory tree, preventing accidental inclusion in searches, linting runs, and IDE indexing. The `claude/` branch prefix identifies the branch as an AI session branch, making it recognizable in the branch list and in GitHub's PR interface.[^1]

Cleanup discipline is the discipline that prevents worktree accumulation. A worktree that is not removed after its session completes remains as a directory on disk and as a registered worktree in git's internal state. Stale worktrees clutter the filesystem, consume disk space, and can interfere with git operations if they reference branches that have been deleted. The worktree removal command — `git worktree remove ../my-repo-claude-session-name` — should be run as the final step of every session's lifecycle.[^3]

**Recommended Practice:**
- Adopt the naming convention `../[repo-name]-claude-[brief-task-description]` for worktree directories and `claude/[brief-task-description]` for their branches. The task description should be two to four words: `claude/auth-refactor`, `claude/postgres-migration`, `claude/api-versioning`. Consistent naming makes worktrees recognizable and auditable.[^4]
- Run `git worktree list` as part of the team's weekly hygiene check. Any worktree more than one sprint old that has not been merged is stale and should be reviewed: either the work is complete and the worktree should be removed, or the work stalled and the situation should be surfaced.[^3]
- Configure `.gitignore` to exclude common worktree sibling patterns if the team uses a consistent naming convention. This prevents accidental inclusion of worktree directories in path-based searches that run from the parent directory.[^4]
- Document the full worktree lifecycle — create, work, review, merge or discard, remove — as a workflow in the team's engineering handbook with copy-pasteable commands. Engineers who need to look up the syntax for worktree operations will do so less reliably than engineers who can run a documented workflow from memory.[^1]

---

## Section 3: The incident.io Pattern

**Description:** The engineering team at incident.io published a detailed account of how they use git worktrees to enable parallel Claude Code sessions safely at the scale of a production engineering team.[^2] Their approach is the clearest existing documentation of worktree-based parallel session management as a production practice, and it directly informed the conventions this memo recommends.

The incident.io pattern has four elements: each Claude Code task runs in its own worktree, always on its own branch; the engineer reviews the worktree's branch as a PR before any merging occurs; multiple tasks can run simultaneously because their worktrees are completely isolated; and worktree lifecycle management is treated as first-class engineering discipline, not administrative overhead. The result of following this pattern was that the team could run three to five parallel Claude Code sessions simultaneously without working tree conflicts, with each session's output being reviewable as an independent unit.[^2]

Their practical setup recommendation is to keep worktrees as siblings to the main repository directory — in a parent directory that contains both the main repo and all active worktrees. This layout makes the relationship between worktrees and the main repository visually clear in the filesystem and simplifies cleanup: the engineer can see all active worktrees by listing the parent directory. A variant that places worktrees in a dedicated `~/worktrees/` directory achieves the same organizational clarity for teams with many repositories.[^1]

**Recommended Practice:**
- Adopt the incident.io convention: every parallel Claude Code session gets its own worktree and its own branch, opened as a PR for review before merge. No exceptions.[^2]
- Maintain worktrees as siblings to the main repository directory for visual clarity. Use a parent directory structure like: `projects/my-repo/` (main clone), `projects/my-repo-claude-auth-refactor/` (session worktree), `projects/my-repo-claude-test-migration/` (second session worktree).[^2]
- For large-scale fan-out migrations — the pattern described in [Workflows/07-agentic-delegation.md](../Workflows/07-agentic-delegation.md) Section 3 — create a worktree for each parallel invocation. Even if the fan-out is scripted with `claude -p`, each invocation should have its own branch so that failures can be isolated and partial results reviewed independently.[^3]
- Share the incident.io blog post as required reading for any engineer who will run parallel Claude Code sessions. Engineers who have read the primary source are more likely to follow the worktree discipline correctly than those who have only encountered the convention in the team's internal documentation.[^2]

---

## Section 4: Worktree Lifecycle Management

**Description:** The worktree lifecycle has four phases: creation before session start, active use during the session, review after session completion, and removal after merge or discard. Each phase has specific requirements that, if not followed, produce either lost work (premature removal) or stale state accumulation (failure to remove). The lifecycle discipline is what separates a team that uses worktrees sustainably from a team that creates them opportunistically and finds itself managing a growing collection of stale branches and orphaned directories.[^3]

Creation is the first control point. A worktree should be created on a branch that is explicitly started from the correct base — typically the latest `main` or the relevant feature branch. Starting a session worktree from a stale or incorrect base produces output that will conflict with main during PR review. The creation command should always be preceded by a `git fetch origin` and explicit `--track origin/main` or equivalent to ensure the worktree base is current.[^4]

Review is the critical quality gate. Claude Code session output in a worktree should be reviewed the same way any branch is reviewed: as a PR, with the writer/reviewer pattern (a fresh Claude session reviews the output), and with human code review. The review happens on the worktree's branch; the worktree itself provides the isolated environment needed to run tests and verify the output before merge. Merging a worktree branch without review is equivalent to merging an unreviewed PR — the worktree does not make review optional, it makes review tractable.[^1]

**Recommended Practice:**
- Always start a session worktree from a current base: run `git fetch origin && git worktree add ../[dir] -b [branch] origin/main` rather than starting from a potentially stale local ref. The cost of starting from a stale base is disproportionate to the cost of the fetch.[^4]
- After session completion, run the test suite from within the worktree directory before opening a PR. Running tests in the worktree context confirms that the session's output is self-consistent and does not depend on any unstaged changes.[^3]
- Open a PR for every worktree branch that contains work intended for merge. Do not merge worktree branches directly to main. The PR is the review gate; bypassing it because the work came from an isolated worktree still bypasses review.[^1]
- Remove the worktree immediately after the PR is merged or the work is discarded. The removal command is `git worktree remove [path]`. If the worktree directory has uncommitted changes, use `git worktree remove --force [path]` only after confirming the changes are either committed to the branch or intentionally discarded.[^4]

---

## Section 5: Integration with Fan-Out Delegation Patterns

**Description:** The fan-out delegation pattern described in [Workflows/07-agentic-delegation.md](../Workflows/07-agentic-delegation.md) generates multiple parallel Claude Code invocations for large-scale migrations and repetitive tasks. Without worktree isolation, fan-out sessions running against the same working directory will conflict — the first session to write a file establishes its state, and subsequent sessions overwrite or are overwritten depending on execution timing. Worktrees convert a fundamentally unsafe parallel execution model into a safe one by giving each fan-out instance an isolated environment.[^1]

The integration between fan-out patterns and worktrees is operationally straightforward but requires scripted setup. A fan-out workflow that creates N parallel sessions must first create N worktrees, assign each session to its worktree, run the sessions, collect results from each worktree branch, and then merge the results through the standard PR review process. The setup script creates the worktrees; the fan-out script runs the sessions; the cleanup script removes completed worktrees after their PRs are merged.[^2]

The review discipline for fan-out output is more demanding than for single-session output because there are N branches to review. The standard pattern is to create a meta-PR that aggregates the fan-out results after individual worktree PRs have been reviewed and approved — a merge branch that combines all fan-out branches before the final merge to main. This allows each parallel output unit to be reviewed independently while maintaining a clean merge history.[^3]

**Recommended Practice:**
- Script worktree creation as part of any fan-out setup. The script should: accept a task list as input, create one worktree per task with a consistent naming convention, output the list of worktree paths and branch names for the fan-out runner to use.[^1]
- For fan-out migrations using `claude -p` loops (as described in Workflows/07-agentic-delegation.md Section 3), pass the worktree directory as the working directory for each `claude -p` invocation using the `--cwd` flag or equivalent. Each invocation operates in its own worktree without any session-level configuration change.[^2]
- After fan-out completion, run tests in each worktree before aggregation. A fan-out instance that fails tests should be held for review rather than included in the aggregation merge. Aggregating passing and failing output produces a result that is harder to diagnose than reviewing the failing instance independently.[^3]
- Document the full fan-out worktree workflow — create N worktrees, run N sessions, review N PRs, aggregate, merge — as a runbook in the team's engineering handbook. Fan-out workflows that are executed from memory produce more errors than those executed from a documented runbook.[^4]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Isolation Discipline | Add no-shared-working-tree rule to CLAUDE.md; educate team on failure mode | Architect |
| Worktree Setup | Adopt naming convention; document full setup commands in engineering handbook | Backend lead |
| incident.io Pattern | Share blog post as required reading; adopt sibling directory convention | Architect |
| Lifecycle Management | Enforce PR review for all worktree branches; remove worktrees after merge | All engineers |
| Fan-Out Integration | Script worktree creation for fan-out workflows; document aggregation runbook | Backend lead |

---

[^1]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    Parallel session patterns; fan-out workflows with `claude -p`; working directory isolation for unattended agent runs; `--allowedTools` scoping for parallel invocations.

[^2]: incident.io — "How We're Shipping Faster with Claude Code and Git Worktrees," incident.io Blog, 2026. https://incident.io/blog/shipping-faster-with-claude-code-and-git-worktrees
    Primary source for the worktree-based parallel session pattern: worktree-per-task discipline, sibling directory layout, PR review requirement for every worktree branch, and the operational outcome of running three to five parallel sessions simultaneously without working tree conflicts.

[^3]: Dave Patten — "The State of AI Coding Agents (2026): From Pair Programming to Autonomous AI Teams," Medium, March 2026. https://medium.com/@dave-patten/the-state-of-ai-coding-agents-2026-from-pair-programming-to-autonomous-ai-teams-b11f2b39232a
    Parallel agent coordination patterns; isolation requirements for fan-out delegation; the review discipline that makes parallel agentic output tractable; lifecycle management as first-class engineering practice.

[^4]: Git — "git-worktree Documentation," Git Reference Manual, 2025. https://git-scm.com/docs/git-worktree
    Complete reference for git worktree commands: `git worktree add`, `git worktree list`, `git worktree remove`, and `git worktree prune`. Branch tracking configuration for worktree creation from remote refs. Force-removal behavior for worktrees with uncommitted changes.

[^5]: Anthropic — "2026 Agentic Coding Trends Report," Anthropic, 2026. https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf
    Parallel delegation as an emerging team practice; the infrastructure prerequisites that distinguish safe parallel execution from uncontrolled concurrency; bounded autonomy requirements for multi-session workflows.
