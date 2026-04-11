## Overview

Session hygiene is the set of practices that govern how engineers manage the lifecycle of individual Claude Code sessions and the system of sessions running concurrently. It is among the most underspecified areas in team AI adoption — often treated as a personal preference rather than a team practice — yet its absence produces predictable quality degradation that compounds over a sprint.

The two core problems session hygiene addresses are context window pollution (sessions that accumulate irrelevant material and degrade in output quality over time) and session fragmentation (parallel work that collides because multiple sessions operate on the same files without coordination). Both are solvable with deliberate practices. Neither resolves itself through AI capability improvements alone.

This memo covers context window management, the git worktree pattern for parallel development, parallel session coordination strategies, session naming and resumption, and the practical resource constraints that bound how many parallel sessions are sustainable.

---

## Section 1: Context Window Degradation

**Description:** Claude's context window holds the entire conversation history — every message, every file read, every command output — from session start. As it fills, Claude's performance degrades in a pattern that is non-obvious to engineers who are not monitoring it. Earlier instructions receive less weight relative to recent context. Architectural decisions established in the first hour of a session may not be reliably applied in the fourth.[^1]

The context window's capacity has grown substantially: Claude Opus 4.6 operates with a 1 million token window, sufficient to hold roughly 25,000 lines of code in a single conversation.[^2] This is large enough that many engineers never consciously hit a limit — but it is not large enough to make context management unnecessary. Long sessions that span multiple tasks, include broad codebase exploration, and accumulate correction cycles can fill a significant fraction of this window with material that actively degrades subsequent output quality.

The most visible symptom is the "correction spiral" — Claude making the same mistake repeatedly because previous failed correction attempts are now part of the context, anchoring it toward the incorrect approach.[^1] Once the spiral starts, continuing the session is slower than clearing and starting fresh with a better-specified prompt. The correction history is not helping; it is a liability.

Automatic compaction occurs when Claude approaches context limits, producing a summary. Engineers can trigger this manually with `/compact [instructions]`, providing explicit guidance about what to preserve. For long sessions, custom compaction instructions in `CLAUDE.md` ensure critical context survives: "When compacting, always preserve the full list of modified files and any test commands."[^1]

**Recommended Practice:**
- Run `/clear` between unrelated tasks rather than continuing in a single long session. Context accumulated from task A actively degrades the quality of task B work in the same session.[^1]
- After more than two failed corrections on the same issue in one session, `/clear` and start fresh with a prompt that incorporates what the corrections taught you. The failed attempts are polluting context, not helping.[^1]
- Use `/compact Focus on [specific area]` for sessions that must continue rather than reset. Provide explicit scope: what needs to be preserved, what can be discarded.[^1]
- Configure a status line to display real-time context window usage. This converts a non-obvious degradation signal into a visible metric that engineers can act on proactively.[^3]
- Set custom compaction instructions in `CLAUDE.md` for your specific codebase: what always needs to be preserved (modified file list, test commands, architectural decisions).[^1]

---

## Section 2: The Git Worktree Pattern

**Description:** Git worktrees allow multiple branches of the same repository to exist simultaneously in separate directories, each with its own working tree, while sharing a common git history and remote. For AI-assisted development, worktrees are the canonical isolation mechanism that enables parallel Claude sessions to work on different features without file conflicts.[^4]

Claude Code's `--worktree` flag creates an isolated worktree and starts Claude within it automatically. Each worktree gets its own directory (`.claude/worktrees/<name>/`) and its own branch. Engineers can run `claude --worktree feature-auth` and `claude --worktree bugfix-123` simultaneously, with each session operating on completely isolated copies of the codebase.[^5]

The incident.io engineering team documented the practical workflow: their custom bash function creates a worktree, checks out a branch, copies necessary environment files, and launches Claude Code in a single command. Their reported outcomes: a JavaScript editor feature estimated at two hours was completed in "30 seconds of prompting and about 10 minutes of processing time," and an API generation optimization yielding an 18% improvement (30 seconds faster on daily runs) cost $8 of Claude credits.[^4]

Worktrees are cleaned up automatically when sessions exit with no changes. If changes exist, Claude prompts to keep or remove the worktree. The `.worktreeinclude` file lists gitignored files (`.env`, `.env.local`, `config/secrets.json`) that should be automatically copied to each new worktree.[^5]

**Recommended Practice:**
- Add `.claude/worktrees/` to `.gitignore` to prevent worktree contents from appearing as untracked files in the main repository.[^5]
- Create a team-shared bash function or shell alias for the common pattern: create worktree + copy env files + launch Claude session. Consistency across engineers reduces setup friction.[^4]
- Use the `--worktree` flag rather than manual git worktree commands when starting new AI feature sessions. Claude handles cleanup automatically on exit.[^5]
- Add a `.worktreeinclude` file listing environment files that need to be copied. Fresh checkouts without environment files fail on first run, interrupting the session.[^5]

---

## Section 3: Parallel Session Coordination

**Description:** Running multiple Claude sessions simultaneously is one of the highest-leverage productivity patterns available, but it requires explicit coordination to avoid conflicts. Boris Cherny runs five parallel sessions as his primary workflow, using system notifications and numbered terminal tabs for coordination. He identifies this as "the single biggest productivity unlock."[^3]

Laurent Kempé's March 2026 documentation of "from 3 worktrees to N" workflows describes the evolution of parallel session patterns: starting with 3 parallel worktrees and scaling upward as coordination processes mature. The key coordination requirement is explicit scope separation: each session must have clearly bounded files and responsibilities, with no session reaching into another session's scope.[^6]

The most common coordination failure is database state collision — two parallel sessions that each modify shared state (a local database, a message queue, a Docker volume) simultaneously. Git worktrees isolate the filesystem, but they do not isolate runtime services. Two Claude sessions both running database migrations against the same local database will produce conflicting state.[^4]

Practical mitigation requires either: using separate databases per worktree (port isolation, Docker network separation), ensuring only one session at a time modifies shared state, or treating all shared-state tasks as sequential rather than parallel. This constraint is real and limits which tasks are appropriate for parallel execution.[^4]

**Recommended Practice:**
- Configure desktop notifications (via the `Notification` hook in settings.json) so that parallel sessions alert when they need input, rather than requiring engineers to actively poll multiple terminals.[^5]
- Establish explicit scope boundaries before launching parallel sessions: which files does each session own? Document these boundaries in the session's opening prompt.[^6]
- Treat database-modifying tasks as sequential, not parallel. Use parallel sessions for filesystem-isolated work (separate modules, separate services, separate feature branches).[^4]
- Use per-session environment variables to configure separate service ports, database names, or API keys when parallel sessions require different infrastructure endpoints.[^4]
- Start with two parallel sessions; add more as coordination overhead proves manageable. Five parallel sessions require a coordination system; two parallel sessions require only file separation.[^3]

---

## Section 4: Session Naming and Resumption

**Description:** Claude Code saves conversations locally and supports resumption across sessions. This transforms the single-session workflow model into a multi-session, multi-day workflow model where context can be preserved across interruptions, meetings, and end-of-day breaks.[^5]

Session naming is the key enabling practice. An anonymous session (default prompt: "fix the login bug") is difficult to locate in a list of recent sessions. A named session (`claude -n oauth-migration`) is immediately identifiable. The `/rename` command names or renames sessions mid-session; named sessions can be resumed by name with `claude --resume oauth-migration`.[^5]

The PR linkage feature extends this: when a PR is created using `gh pr create`, the session is automatically linked to that PR. The PR can be resumed later with `claude --from-pr <number>`, restoring the full session context for review responses, additional changes, or post-merge follow-up.[^5]

Sessions can also be branched. The `/branch` command or `--fork-session` flag creates a fork of the current session for exploring an alternative approach without losing the original session's state. This enables safe experimentation: try the risky refactor in a forked session; if it works, continue; if not, discard the fork and return to the original session.[^5]

**Recommended Practice:**
- Name every non-trivial session at start with `-n session-name` using the feature or task as the name. "oauth-migration" and "fix-auth-timeout" are recoverable; "my session" and the session ID are not.[^5]
- Establish a team naming convention. A consistent format (`component-action`, e.g., `auth-oauth-migration`, `payments-refactor-stripe`) makes sessions findable for any team member who needs to continue the work.[^4]
- Use `--from-pr` when returning to continue PR review feedback. This restores the session context that produced the PR rather than starting fresh with only the diff as context.[^5]
- Use session forking (`/branch`) before attempting risky refactors or exploratory changes. The fork is a safe experiment; the original session is the fallback.[^5]

---

## Section 5: Resource Constraints in Parallel Work

**Description:** Parallel sessions consume resources at the model tier level (API rate limits, usage limits), at the machine level (CPU, memory, available terminal panes), and at the cognitive level (engineer's attention across multiple concurrent workstreams). All three constraints are real and interact.[^3]

Model usage limits apply per account. Heeki Park's documented experience found that Opus 4.6 hit usage limits within 45 minutes to one hour of heavy use, while Sonnet 4.6 sustained several hours without limits.[^7] For parallel sessions, this limit is shared — five sessions all using Opus 4.6 simultaneously exhaust the limit proportionally faster than one session. Teams should calibrate parallel session counts against model tier, using Sonnet 4.6 for sustained parallel work and reserving Opus 4.6 with thinking mode for high-value planning sessions.[^7]

Attention is the binding constraint for most engineers. Two focused parallel sessions — each working on a well-defined, bounded task — produce more total output than five sessions each requiring frequent steering. The cognitive overhead of coordinating five simultaneous AI workstreams is substantial, and context-switching between them produces the same comprehension risk as parallel human multitasking.[^3]

The Fazm blog's documentation of Claude Code parallel session patterns identifies "notification fatigue" as a practical failure mode: when all sessions surface questions simultaneously, the engineer provides lower-quality responses to all of them than they would to one at a time. Staggering session starts so they don't all require input simultaneously is a practical mitigation.[^8]

**Recommended Practice:**
- Calibrate parallel session count to attention budget, not machine resources. Two focused sessions with good task definitions generally outperform five sessions with loose definitions.[^3]
- Use Sonnet 4.6 for parallel implementation sessions; reserve Opus 4.6 with thinking mode for planning and complex reasoning tasks. This manages usage limits while allocating reasoning capacity where it has the highest return.[^7]
- Stagger session starts by 5-10 minutes when launching multiple parallel sessions. This prevents simultaneous input requests and distributes cognitive load more evenly.[^8]
- Track which task types are most productive in parallel (isolated filesystem work, long-running generation tasks) vs. sequential (shared-state modification, architectural decisions, security-critical paths). Build a team-specific parallel-eligible task list over time.[^6]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Context Window Management | Configure /compact; set CLAUDE.md compaction instructions; /clear between tasks | All engineers |
| Git Worktrees | Add .claude/worktrees/ to .gitignore; create team bash alias for worktree creation | All engineers |
| Parallel Session Coordination | Set explicit scope boundaries; treat database tasks as sequential | All engineers |
| Session Naming | Establish team naming convention; use -n at session start | All engineers |
| Resource Management | Use Sonnet 4.6 for parallel work; calibrate to attention budget, not machine capacity | All engineers |

---

[^1]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    - Context window degradation patterns: kitchen sink session, correction spiral, performance impact
    - /clear, /compact, and /rewind as the primary context management tools
    - Custom compaction instructions in CLAUDE.md for preserving critical context across resets

[^2]: Claude Directory — "Claude Opus 4.6 and the Million-Token Context Window: What It Changes for Developers," 2026. https://www.claudedirectory.org/blog/claude-opus-4-million-token-era
    - 1 million token context window: ~25,000 lines of code capacity
    - Context window growth over model generations and its practical implications for session length
    - Context management discipline remains essential even with large windows

[^3]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    - Five parallel sessions as the "single biggest productivity unlock": numbered tabs, system notifications for coordination
    - Opus 4.6 with thinking mode: less steering requirement despite larger, slower model
    - Context window status line for real-time token usage tracking

[^4]: incident.io — "How We're Shipping Faster with Claude Code and Git Worktrees," incident.io Blog, 2026. https://incident.io/blog/shipping-faster-with-claude-code-and-git-worktrees
    - Custom bash function for worktree creation + Claude session launch in one command
    - Productivity outcomes: 2-hour feature in "30 seconds of prompting, 10 minutes processing"; 18% API improvement for $8 of credits
    - Database state collision as the primary parallel session failure mode; port isolation mitigation

[^5]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    - --worktree flag, .worktreeinclude file, automatic cleanup behavior
    - Session naming with -n, /rename, and --resume session-name
    - PR linkage and --from-pr resumption; session forking with /branch

[^6]: Laurent Kempé — "From 3 Worktrees to N: How AI Agents Changed My Parallel Development Workflow on Windows," March 2026. https://laurentkempe.com/2026/03/31/from-3-worktrees-to-n-ai-powered-parallel-development-on-windows/
    - Scaling parallel worktrees from 3 to N: coordination process maturity as the bottleneck
    - Explicit scope boundaries as the key coordination requirement
    - Task categorization: which work is parallel-eligible vs. sequentially required

[^7]: Heeki Park — "Using Spec-Driven Development with Claude Code," Medium, March 2026. https://heeki.medium.com/using-spec-driven-development-with-claude-code-4a1ebe5d9f29
    - Opus 4.6 usage limit: 45 minutes to 1 hour under heavy parallel use
    - Sonnet 4.6 sustained several hours in the same workflow
    - Context compaction timing: 3–5 minutes optimistic, 10–12 minutes heavy usage

[^8]: Fazm Blog — "Claude Code Parallel Sessions: How to Run Multiple Agents at Once," 2026. https://fazm.ai/blog/claude-code-parallel-sessions
    - Notification fatigue: simultaneous input requests from parallel sessions degrading response quality
    - Session staggering as a practical coordination technique
    - Per-session environment variable configuration for isolated service endpoints

[^9]: Dogukan Uraz Tuna — "Mastering Git Worktrees with Claude Code for Parallel Development Workflow," Medium, 2026. https://medium.com/@dtunai/mastering-git-worktrees-with-claude-code-for-parallel-development-workflow-41dc91e645fe
    - Git worktree setup and parallel Claude session management in practice
    - Shared .git history while maintaining isolated working directories per session
    - Coordination patterns for managing multiple simultaneous AI workstreams

[^10]: MindStudio — "What Is the Claude Code Git Worktree Pattern? How to Run Parallel Feature Branches With AI," 2026. https://www.mindstudio.ai/blog/what-is-claude-code-git-worktree-pattern-parallel-feature-branches
    - Git worktree pattern explained for teams new to the approach
    - Parallel feature branch isolation: preventing session cross-contamination
    - Merge strategy for parallel worktree outputs

[^11]: Nimbalyst — "Coding with AI Agents: Best Practices for 2026," 2026. https://nimbalyst.com/blog/coding-with-ai-agents-best-practices-2026/
    - Session management discipline as a key differentiator between high-performing and low-performing AI-assisted teams
    - Attention budget as the real constraint on parallel session count
    - Tool integrations (tmux, terminal multiplexers) for managing multiple concurrent AI sessions
