## Overview

The Explore-Plan-Implement loop is the foundational workflow pattern for non-trivial AI-assisted development. It is not a rigid ceremony but a cognitive discipline: the habit of separating understanding from action, planning from execution, and exploration from commitment. On a team under delivery pressure, this separation is the first thing to erode — and its erosion is the primary cause of the rework, architectural drift, and comprehension debt described in the Issues section.

This memo goes deeper than the overview's four-phase summary. It covers how Plan Mode works mechanically, how extended thinking changes the planning phase, when the loop should be skipped, and the specific failure modes that emerge when teams shortcut it.

---

## Section 1: What Plan Mode Actually Does

**Description:** Plan Mode in Claude Code is a read-only execution environment activated by pressing Shift+Tab twice or by launching with `--permission-mode plan`. In this mode, Claude analyzes codebases using only read operations — it cannot write files, run commands, or make changes of any kind. The constraint is not advisory; it is enforced at the tool level. Claude can read files, search, and ask clarifying questions using `AskUserQuestion`, but the only output is analysis and a proposed plan.[^1]

This distinction matters more than it may appear. In normal mode, Claude's tendency is to produce output — to write code, to make changes, to demonstrate forward progress. Plan Mode removes this incentive structure entirely. Claude is forced to think before it acts, which produces materially different results: implementations that account for the actual shape of the existing codebase rather than what the AI assumes the codebase looks like based on the prompt.

The output of a Plan Mode session is an implementation plan in structured text. Engineers can open this plan directly in their text editor using Ctrl+G before any implementation begins, edit it, challenge assumptions in it, and only then approve execution. This editing step is not optional overhead — it is the point at which the engineer's contextual knowledge (the things Claude cannot know from reading files alone) enters the process. A plan that requires revision is faster than debugging incorrect code post-implementation.[^2]

**Recommended Practice:**
- Activate Plan Mode by default for any task involving more than two files or unfamiliar code. Configure it as the session default in `.claude/settings.json` for engineers working in new areas of the codebase.[^1]
- Always open the generated plan in an external editor with Ctrl+G before approving it. Read it for architectural assumptions, not just correctness — the most dangerous planning mistakes are coherent but wrong.[^3]
- When Plan Mode surfaces ambiguous decisions (authentication patterns, data model choices, API design), treat these as signals for a human architectural conversation, not prompts to accept the AI's default choice.[^4]
- Use the headless variant `claude --permission-mode plan -p "..."` in CI pipelines to generate analysis artifacts — architecture summaries, migration plans, dependency audits — without any risk of automated changes.[^1]

---

## Section 2: The Four Phases in Practice

**Description:** The full Explore-Plan-Implement-Commit sequence is the documented workflow from Anthropic's internal teams. Each phase has a distinct objective and failure mode. The Explore phase establishes shared understanding between the engineer and Claude. The Plan phase produces the implementation contract. The Implement phase executes against that contract. The Commit phase creates a verifiable checkpoint.[^1]

In practice, the most commonly skipped phase is Explore. Engineers with familiarity with the codebase often skip directly to Plan, providing context in the prompt rather than having Claude read files. This produces faster sessions but higher risk: Claude's internal model of the codebase is built from the prompt description, not from reading the actual files, and the two diverge in ways that produce implementation bugs. Addy Osmani, in his documented workflow for 2026, describes the Explore phase as "due diligence" — and notes that "upfront planning pays dividends for implementation efficiency and output quality" even when it feels redundant.[^5]

The Commit phase is frequently treated as a housekeeping step rather than a workflow phase. Treating it as a genuine checkpoint — committing after each successful implementation task with a descriptive message, rather than accumulating multiple AI changes into one large commit — creates a safety net that enables session resets, rollbacks, and debugging trails that are otherwise unavailable.[^2]

A completed spec-driven project using this workflow at scale has demonstrated the pattern's leverage: one engineer completed a SQLite-to-IndexedDB migration in one afternoon using 14 sequential atomic commits, each corresponding to a delegated subagent task. The same work was estimated at 2-3 days manually.[^6]

**Recommended Practice:**
- Structure each session explicitly: begin with "Read [files] and understand [specific thing]" before any implementation prompt. Do not skip Explore even when you believe you understand the codebase.[^5]
- Use the Plan phase to surface design ambiguities before they become implementation bugs. If Claude's plan includes an assumption you disagree with, correct it in the plan editor rather than in a post-implementation correction.[^1]
- Treat each completed task — not each session — as a commit boundary. Small, frequent commits enable session resets without work loss.[^2]
- After accepting a plan, Claude automatically names the session from the plan content, making it retrievable later with `--resume`.[^1]

---

## Section 3: When to Plan, When to Skip

**Description:** Not every task benefits from the Explore-Plan-Implement loop. Applying it uniformly to all tasks creates unnecessary overhead and trains engineers to resent the process. The discipline is in knowing which tasks require it — and being honest when time pressure is causing a skip that should not happen.

Anthropic's own guidance identifies the conditions where planning is most valuable: multi-file changes, unfamiliar code, and tasks where the correct implementation approach is genuinely uncertain.[^1] Tasks that can be described as a single sentence diff — rename this variable, add a log line, change this return type — do not benefit from a planning phase. For these, the planning overhead costs more than it saves.

The counter-risk is the "scope creep at planning time" problem: a task that appears simple in the prompt turns out to touch multiple files once exploration begins. Plan Mode's read-only Explore phase is the appropriate mechanism for this discovery — it surfaces the true scope of a task before any changes are made, allowing the engineer to either narrow the task or consciously accept the broader scope. This is significantly better than discovering scope creep mid-implementation when context is already polluted and rollback is costly.[^3]

Google AI Director Addy Osmani's 2026 workflow documentation describes three levels of specification: product-level specs (long-term vision), change-level specs (specific feature), and implementation plans (step-by-step execution). The Explore-Plan-Implement loop maps directly onto change-level spec creation and subsequent implementation plan execution.[^5]

**Recommended Practice:**
- Establish a team heuristic: if the task cannot be described as a one-sentence diff, it requires a planning phase. If it could change files you haven't thought of yet, it requires an Explore phase.[^1]
- When time pressure causes a planning skip, make the skip explicit and intentional — not habitual. "We are skipping planning because X" is a different posture than "we never plan." The former preserves the discipline; the latter erodes it.[^4]
- Use Plan Mode for code exploration and onboarding tasks as well, not just implementation. "Read the authentication module and produce a summary of how session management works" is a valuable use of the read-only Explore phase.[^1]
- Introduce task tagging in sprint planning: classify tasks as "plan-required" vs. "direct" so the overhead expectation is set before the session begins.[^4]

---

## Section 4: Extended Thinking and Effort Levels

**Description:** Opus 4.6 and Sonnet 4.6 support adaptive reasoning — the model dynamically allocates thinking tokens based on an effort level setting rather than a fixed budget. This changes how the planning phase should be calibrated: high-effort sessions with extended thinking produce more thorough plans for complex architectural decisions, while low-effort sessions produce faster plans for routine tasks.[^1]

The `ultrathink` keyword instructs Claude to allocate maximum thinking tokens for a single turn without changing the session's default effort level. For the planning phase of a complex architectural refactor, this is a direct mechanism for getting a deeper plan without committing to higher-effort mode for the entire session.[^1]

Boris Cherny's documented workflow uses Opus 4.6 with thinking mode for all work, arguing that "less steering plus better tool use equals faster overall results, even with a larger model."[^7] For a team managing costs, a pragmatic middle ground is using higher-effort settings for the Plan phase and standard settings for implementation — the planning phase is where reasoning depth has the highest return.

**Recommended Practice:**
- Set effort level to high (`/effort high` or via model config) for the Explore and Plan phases of significant features. Return to standard effort for implementation, where speed matters more than reasoning depth.[^1]
- Use `ultrathink` in the Plan phase for complex architectural decisions without committing to high effort for the whole session.[^1]
- Enable verbose mode (Ctrl+O) during the planning phase to inspect Claude's reasoning. Visible reasoning catches planning mistakes before they become implementation mistakes.[^1]
- Note that thinking tokens are billed even when summaries are redacted; set `showThinkingSummaries: true` in settings.json during team review sessions so planning reasoning is visible to all reviewers.[^1]

---

## Section 5: Failure Modes and Recovery

**Description:** The most common failure modes in the Explore-Plan-Implement loop are identifiable and correctable. Recognition is the first step.

The **wrong-problem plan** occurs when Claude produces a correct plan for the wrong interpretation of the requirement. The mitigation is the Explore phase followed by explicit review of the plan's stated goal before approving implementation. If the plan says "add OAuth support" but the requirement was "add OAuth login for the admin portal only," the plan is wrong even if technically coherent.[^1]

The **plan drift problem** occurs when implementation deviates from the approved plan mid-session as Claude encounters unexpected complexity. The mitigation is to ask Claude to compare its current implementation state against the original plan periodically, or to run a separate reviewer session (see Workflow 6) that evaluates the output against the specification.[^8]

The **overcorrection problem** occurs when engineers apply the loop to tasks that do not warrant it, creating overhead that generates resentment and eventual workflow abandonment. The mitigation is clear team criteria for when the loop applies.[^1]

METR's February 2026 update on their developer productivity studies found that a significant portion of developers now refuse to complete coding tasks without AI assistance — a finding that underscores both the tool's value and the importance of structured workflows that preserve human judgment even as AI takes on more execution.[^9]

**Recommended Practice:**
- When a mid-implementation session deviates from its plan, stop and assess before continuing. Compare the current state against the approved plan explicitly: "Does what you've implemented so far match the plan we approved? What diverged?"[^1]
- For sessions that have gone significantly off-plan, use `/rewind` to restore conversation and code state to the plan approval point and restart implementation with a revised prompt.[^1]
- After a failed session (incorrect implementation requiring significant rework), document the planning failure and update `CLAUDE.md` with the correction. The CLAUDE.md should capture not just architectural rules but workflow corrections that prevent recurrence.[^7]
- Conduct a brief post-mortem on any PR that required more than two rounds of implementation correction: was the plan phase skipped? Was the plan not edited carefully enough? Was the task scope too large?[^4]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Activate Plan Mode | Set plan mode default for all new-area work in settings.json | All engineers |
| Edit Plans Before Approving | Open with Ctrl+G; review for architectural assumptions, not just correctness | All engineers |
| Commit at Task Boundaries | Treat each completed task as a commit, not each session | All engineers |
| Skip Criteria | Establish team heuristic: single-sentence diff = skip; multi-file = plan required | Architect |
| Extended Thinking | Use high effort / ultrathink for planning phase of complex features | All engineers |
| Plan Drift Recovery | Compare implementation against plan at session midpoints for long tasks | All engineers |

---

[^1]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    - Plan Mode mechanics: read-only constraints, Ctrl+G to edit plans, headless plan mode for CI
    - Session naming from accepted plans; worktree integration; `--continue` and `--resume` flags
    - Extended thinking and effort levels: ultrathink keyword, adaptive reasoning in Opus 4.6 and Sonnet 4.6

[^2]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    - Explore-Plan-Implement-Commit as the four-phase recommended workflow; when to skip planning
    - Commit discipline: frequent commits as save points; session resets without work loss
    - Common failure patterns: kitchen sink session, correction spiral, over-specified CLAUDE.md

[^3]: Claude Code Plan Mode Guide — "Claude Code Plan Mode: Complete Guide (2026)," GetAIPerk, 2026. https://www.getaiperks.com/en/articles/claude-code-plan-mode
    - Scope discovery: how Plan Mode surfaces true task scope before implementation begins
    - Typical four-phase workflow with follow-up refinement questions
    - Backward compatibility and database migration examples

[^4]: Developer Workflows with AI Tools (2026) — Vibecoding.app, 2026. https://vibecoding.app/blog/developer-workflows-with-ai
    - Planning-first philosophy: brainstorm spec → outline plan → write code sequence
    - Sprint planning integration: when to classify tasks as plan-required vs. direct
    - Team norm setting: making planning skips explicit rather than habitual

[^5]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    - Three-tier spec hierarchy: product-level, change-level, and implementation plans
    - "Waterfall in 15 minutes": rapid structured planning that prevents wasted downstream cycles
    - AI-augmented engineering philosophy: "The human engineer remains the director"

[^6]: Artur Less — "Spec-Driven Development with Claude Code," Level Up Coding / Medium, March 2026. https://levelup.gitconnected.com/spec-driven-development-with-claude-code-1b08184965e3
    - Four-phase spec workflow: parallel research → spec creation → interview refinement → task delegation
    - 14 atomic commits across 15+ files; SQLite-to-IndexedDB migration completed in one afternoon vs. 2–3 days estimated manually
    - Context usage: 71% (143k/200k tokens) despite coordinating multiple subagents

[^7]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    - Opus 4.6 with thinking mode rationale: less steering + better tool use = faster net results
    - Plan-first approach: iterating on plans before auto-accept to achieve "1-shot implementations"
    - CLAUDE.md discipline: updating the file when Claude makes errors, not just setting it and forgetting

[^8]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
    - Plan drift problem: comparing implementation state against original specification as a CI gate
    - Staged writes pattern: patching as files before application rather than direct modification
    - Validation layer separating review output from write operations

[^9]: METR — "We Are Changing Our Developer Productivity Experiment Design," METR Research, February 2026. https://www.metr.org/blog/2026-02-24-uplift-update/
    - Growing developer dependency on AI: significant portion of developers declining tasks they cannot complete with AI
    - Selection bias emerging in productivity studies: developers avoiding structured task assignment
    - Implication: workflows that preserve human judgment under AI assistance are increasingly critical to measure and maintain

[^10]: VS Code Planning Mode — "VS Code Planning Mode: Think Before You Code," OpenReplay Blog, 2026. https://blog.openreplay.com/vs-code-planning-mode/
    - Cross-tool adoption of the plan-before-act pattern across Claude Code, Codex, and VS Code
    - Pattern generalization: the same Explore-Plan-Implement structure emerging independently across AI coding tools
    - Read-only constraint design: why the mechanical enforcement of planning matters more than advisory guidelines

[^11]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
    - Plan-first as the foundation of "1-shot implementation": how iterating on plans before execution eliminates correction cycles
    - Building at Anthropic: teams building for the model "six months from now," not today's limitations
    - Parallel session strategy as a complement to the planning loop for larger features
