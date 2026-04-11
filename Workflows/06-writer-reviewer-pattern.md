## Overview

The writer/reviewer pattern is the practice of separating AI code generation from AI code review into two independent sessions with no shared context. It addresses a structural bias that emerges when the same agent — or the same human — both writes and reviews code: the reviewer's knowledge of the implementation choices they made biases them toward rationalizing rather than scrutinizing those choices.

This is not a new problem. Human software teams have understood for decades that self-review is unreliable. What is new is that AI-assisted development creates a version of this problem at scale: when AI generates most of the code and the same session is asked to review it, the review is structurally compromised. The pattern is in the code, not in anyone's head — but the AI session that generated it has the same insider-knowledge bias that makes human self-review unreliable.

This memo covers the mechanics of self-review bias, the implementer/reviewer architecture, specialist-agent review patterns, CI-integrated agentic review, and how to route reviewed output to human reviewers based on risk level.

---

## Section 1: The Mechanics of Self-Review Bias

**Description:** When an AI session generates code, it makes implicit architectural decisions, selects specific implementation approaches, and frames problems in particular ways. These decisions are embedded in the session's context window — not just as code, but as the reasoning that led to the code. When the same session is asked to review the output, it reviews it through the lens of the decisions it already made. It is structurally incapable of seeing the code with fresh eyes.[^1]

The practical effect is that self-review by the generating session catches surface-level errors (syntax, obvious bugs, missing null checks) while consistently missing structural problems (wrong abstraction, inconsistent with existing patterns, edge cases the generator didn't consider, architectural decisions that contradict the team's conventions). These are precisely the issues that matter most — and the issues that Issue 1 (Comprehension Debt) warns will compound as AI code generation increases.

This is documented across multiple 2026 analyses of AI code review workflows. The characterization from the manasight.gg analysis is precise: "The agent session that wrote the code tends to be biased toward its own decisions, so a separate agent is invoked to check. That separation matters more than it sounds. When an agent generates code, it's 'warm' — it has all the context, it knows what it intended. It's hard for it to spot its own mistakes. But a fresh reviewer agent sees the code with new eyes."[^1]

The operational implication is that fresh-context review is not a luxury for high-stakes code — it is a baseline requirement for AI-generated code at any scale. A 300-line AI-generated PR reviewed only by the generating session may have caught zero of the structural problems it contains.

**Recommended Practice:**
- Establish the rule: no AI-generated PR is reviewed only by the generating session. Every non-trivial AI PR receives at minimum a fresh-context reviewer session before human review.[^1]
- Explain the bias to the team in concrete terms: the generating session knows what it intended; the reviewer session knows only what the code says. These are different and complementary perspectives.[^2]
- Use the fresh-context review output as the pre-read for human reviewers: share the reviewer session's findings as a comment on the PR before human review begins. This focuses human attention on the structural concerns the reviewer raised, not on re-discovering them.[^3]
- Make the rule explicit in `CLAUDE.md` as a workflow requirement, not just a guideline. This ensures it applies consistently across engineers regardless of time pressure.[^4]

---

## Section 2: The Implementer/Reviewer Architecture

**Description:** The implementer/reviewer architecture deploys two Claude sessions sequentially for each significant piece of work: one session (the implementer) owns all code generation; a separate session (the reviewer) owns all review of that code with fresh context, no knowledge of the implementation choices made, and no bias toward them.[^1]

The reviewer session should be configured with three inputs: the code to review (via `@` file references), the `CLAUDE.md` architectural context, and the relevant specification or acceptance criteria. A reviewer that does not know the team's conventions cannot catch violations of them. A reviewer that does not know the acceptance criteria cannot catch code that passes tests but fails requirements.[^4]

Qodo's 2026 AI code review pattern analysis identifies the evolution toward specialist-agent review: rather than one reviewer session evaluating all dimensions of the code, multiple specialized agents each evaluate a specific dimension — correctness, security, performance, observability, standards compliance.[^5] Each specialist has focused expertise and a narrower evaluation scope, producing higher-quality findings in each domain. A coordinator agent then consolidates findings into coherent, prioritized feedback.

The practical result of this pattern, as documented in the manasight.gg analysis: "Agents reviewing from fresh context found problems the implementation agent missed, and the PR comments felt like having another person on the team."[^1]

**Recommended Practice:**
- Configure the reviewer session with the output of the implementer session, the `CLAUDE.md`, and the spec.md. The reviewer needs all three to evaluate architecture, conventions, and requirements simultaneously.[^4]
- For security-critical or architecturally significant PRs, deploy a security-specialist reviewer session separately from the general reviewer session. Different expertise, separate context, separate findings.[^5]
- Share the reviewer session's full output as a PR comment before the human reviewer begins. Format it with severity levels: what blocks merge, what should be addressed, what is optional.[^5]
- Do not show the implementer session the reviewer's feedback and ask it to defend its choices. This defeats the separation. Start a fresh implementer session for revisions, seeded with the reviewer's findings as context.[^2]

---

## Section 3: Specialist-Agent Review

**Description:** The five specialist-agent review patterns identified in Qodo's 2026 analysis represent a maturation of the simple writer/reviewer split into a structured review architecture. Each pattern addresses a specific failure mode in AI-generated code review.[^5]

**Context-First Review** assembles the review context — cross-repo usages, historical PRs, architecture documentation — before any evaluation begins. This is the prerequisite for meaningful architectural review of AI-generated code; without context, the reviewer can only evaluate local correctness, missing system-level impacts.[^5]

**Severity-Driven Review** organizes findings into three tiers: Action Required (blocks merge), Recommended (should be addressed), and Minor Suggestions (optional). This prevents critical security or correctness issues from being buried under cosmetic feedback — a pattern that emerges when AI reviewers produce long lists of findings without prioritization.[^5]

**Specialist-Agent Review** deploys separate agents for correctness, security, performance, observability, requirements compliance, and coding standards. Each agent has a narrower scope and produces higher-quality findings in its domain than a single generalist agent would. A coordinator consolidates.[^5]

**Attribution-Based Review** tracks which suggestions are accepted, modified, dismissed, or ignored. Over time, this creates a learning loop: consistently-dismissed suggestion categories are downgraded; consistently-accepted ones are weighted more heavily. For a team that runs many AI-generated PRs, attribution feedback compounds into progressively better review quality.[^5]

**Flow-to-Fix Review** connects reviewer findings directly to generated fix candidates, allowing engineers to move from finding to suggested patch to application in a single workflow without breaking out of the review context.[^5]

**Recommended Practice:**
- Start with severity-driven review: require all reviewer sessions to categorize findings as Action Required / Recommended / Optional. This alone significantly improves the signal-to-noise ratio of AI review output.[^5]
- Assign the architect as the operator of the context-first review session for architecturally significant PRs. The context assembly step requires organizational knowledge that the reviewer session cannot generate on its own.[^3]
- Over time, track which reviewer session findings are accepted vs. dismissed by the team. Use this data to refine reviewer session prompts and prioritize finding categories.[^5]
- For the security specialist agent, use the subagent definition approach from the best practices documentation: a `.claude/agents/security-reviewer.md` with focused expertise, restricted tool access, and a system prompt emphasizing injection vulnerabilities, auth flaws, and credential handling.[^4]

---

## Section 4: CI-Integrated Agentic Review

**Description:** Integrating agentic review into CI pipelines creates a scalable version of the writer/reviewer pattern that operates on every PR without requiring engineer overhead per review. The CI-integrated reviewer runs automatically, produces findings before the human reviewer begins, and blocks merge for Action Required findings.[^6]

Roman Fedytskyi's March 2026 documentation of safer CI patterns for agentic code review provides the architecture: review runs in a scoped, read-only environment; findings are validated against engineering and policy gates before being published; writes are staged as patch files rather than applied directly; and full audit trails log all prompts, tool calls, and validation results.[^6]

The safety model is important: CI-integrated agents that can modify code or take write actions in production environments introduce significant risk. The pattern separates review (read-only, findings emitted as artifacts) from remediation (write actions requiring human approval or a higher-trust environment). This preserves the value of automated review while maintaining a human gate on all code changes.[^6]

For our team, a practical starting point is the "Claude as linter" pattern in Claude Code's headless mode: `claude -p "review the changes vs. main and report issues with severity levels"` added to the build script. This produces structured review output on every PR with zero per-PR overhead.[^4]

**Recommended Practice:**
- Add a `claude -p "lint and review changes vs. main"` step to the CI pipeline, configured to output structured findings in JSON format for downstream consumption.[^4]
- Separate CI review steps by concern: one step for code correctness, one for security, one for architectural pattern compliance. This mirrors the specialist-agent approach and produces more actionable output.[^5]
- Configure the CI review agent with read-only tool access and output-to-artifacts permissions. The agent should never have write access to the repository in CI.[^6]
- Use CI review findings as the pre-read for human reviewers: surface the agent's findings in the PR comment thread automatically so human reviewers can focus on addressing flagged issues rather than discovering them.[^3]

---

## Section 5: Human Handoff and Risk Routing

**Description:** The writer/reviewer pattern is not a replacement for human review — it is a preprocessing step that makes human review faster, more focused, and more effective. The goal is to use AI review to eliminate the low-level finding work (syntax errors, missing null checks, obviously incorrect patterns) so that human reviewers can focus on judgment-intensive work: architectural soundness, product correctness, security risk assessment, and comprehension verification.[^2]

Risk routing is the practice of applying different levels of human review intensity to different PR categories based on risk profile. Infrastructure and security changes get closest human scrutiny. Frontend and utility changes get lighter passes.[^1] For AI-generated code specifically, the risk profile is higher than for human-written code (Issue 4 documents that AI-generated code is 2.74× more prone to security vulnerabilities), which shifts every AI-generated PR toward a higher review intensity tier.

The comprehensive human review model for AI-generated PRs — one that accounts for both the writer/reviewer pattern output and the human reviewer's role — has three steps: (1) review the AI reviewer's findings and verify they are addressed; (2) verify the engineer can explain the implementation (the "can you explain it" merge rule from Issue 1); and (3) evaluate architectural fit against the `CLAUDE.md` conventions and the current sprint's design decisions.[^7]

**Recommended Practice:**
- Route AI-generated PRs to a higher review intensity tier than human-written PRs as a baseline. The statistical risk profile justifies it.[^7]
- Use the "can you explain it" merge rule for AI-generated code: before approving, the reviewer should be able to elicit a coherent explanation of the implementation from the author. Code the author cannot explain should not merge.[^7]
- Structure PR descriptions for AI-generated code to include: what the AI implemented, what the reviewer session flagged, and what was changed in response. This provides reviewers with the full context of the AI session lifecycle, not just the final diff.[^2]
- For the architect: maintain a list of architectural patterns that AI consistently gets wrong for your codebase. Add these as explicit checks to the reviewer session prompt, so AI review catches AI-specific failure modes in your stack.[^4]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Fresh-Context Review Requirement | Establish rule: no AI PR reviewed only by generating session | Architect |
| Implementer/Reviewer Architecture | Configure standard reviewer session with CLAUDE.md + spec.md inputs | All engineers |
| Specialist Agent Definitions | Create security-reviewer.md in .claude/agents/ | Backend lead |
| Severity-Driven Findings | Require Action Required / Recommended / Optional in all reviewer output | All engineers |
| CI Integration | Add claude -p review step to CI pipeline (read-only, output to artifacts) | Backend lead |
| Human Handoff | Apply "can you explain it" rule; route AI PRs to higher review tier | Architect |

---

[^1]: Manasight Blog — "Everyone Writes About AI Generating Code. Nobody Writes About AI Testing It," 2026. https://blog.manasight.gg/ai-testing-implementer-reviewer-pattern/
    - Fresh-context bias mechanics: "the agent that wrote it is 'warm' — it knows what it intended; the reviewer sees only what the code says"
    - Implementer/reviewer pattern in practice: multi-agent pipeline with fresh-context review
    - "PR comments felt like having another person on the team": practical quality improvement from the pattern

[^2]: Collin Wilkins — "AI Code Review: Approaches, Tools, and Best Practices (2026)." https://collinwilkins.com/articles/ai-code-review-best-practices-approaches-tools.html
    - Self-review bias mechanics and the structural case for fresh-context review
    - PR description structure for AI-generated code: lifecycle transparency for human reviewers
    - Best practice framework for AI code review in 2026

[^3]: AI Tools for Developers 2026 — Cortex, 2026. https://www.cortex.io/post/the-engineering-leaders-guide-to-ai-tools-for-developers-in-2026
    - Using AI review findings as pre-read material for human reviewers
    - Architect-as-context-provider for architectural review of AI-generated code
    - Risk routing framework: different review intensities for different PR categories

[^4]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    - Security reviewer subagent definition: `.claude/agents/security-reviewer.md` pattern
    - CLAUDE.md as reviewer context input: conventions the reviewer must know to catch violations
    - CI integration: `claude -p "lint changes vs. main"` as build script component

[^5]: Qodo — "5 AI Code Review Pattern Predictions in 2026," 2026. https://www.qodo.ai/blog/5-ai-code-review-pattern-predictions-in-2026/
    - Five patterns: Context-First, Severity-Driven, Specialist-Agent, Attribution-Based, Flow-to-Fix
    - Severity tiers: Action Required / Recommended / Optional — preventing critical findings from being buried
    - Attribution-based learning: tracking accepted vs. dismissed suggestions to improve review quality over time

[^6]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
    - Four-layer CI architecture: scope → context → validation → observability
    - Read-only agent access in CI; staged patch output rather than direct writes
    - Human approval gate for all write operations; audit trails for all agent actions

[^7]: Issues Overview — "Issue 1: Comprehension Debt" and "Issue 5: Review Theater," ClaudeCodeReview. Issues/overview.md
    - "Can you explain it" merge rule as the human verification step for AI-generated code
    - AI-generated code is 2.74× more prone to security vulnerabilities: statistical basis for elevated review intensity
    - Separating architectural review from implementation review as complementary human-led steps

[^8]: BSWEN Documentation — "Best Practices for Reviewing AI-Generated Code: Catch Subtle Mistakes Before They Compound," April 2026. https://docs.bswen.com/blog/2026-04-09-ai-code-review-best-practices/
    - Systematic review checklist for AI-generated code: categories of subtle mistakes to check
    - Why AI-generated code compounding errors is structurally different from human error accumulation
    - Practical review workflow for engineering teams with high AI code generation rates

[^9]: Allthingsai.org — "Orchestrate Agentic AI: Context, Checklists, and No-Miss Reviews," 2026 All Things AI Conference. https://2026.allthingsai.org/sessions/orchestrate-agentic-ai-context-checklists-and-no-miss-reviews
    - Checklist-driven agentic review: ensuring no review category is missed through structured review prompts
    - Context assembly as a prerequisite to meaningful review
    - No-miss review architecture for high-volume AI code generation

[^10]: Augment Code — "10 Open Source AI Code Review Tools Tested on a 450K-File Monorepo [2026 Rankings]," 2026. https://www.augmentcode.com/tools/open-source-ai-code-review-tools-worth-trying
    - Comparative analysis of AI code review tools at scale
    - Fresh-context review tools vs. integrated session review: performance differences at monorepo scale
    - Tool selection criteria for teams with high AI code generation rates

[^11]: Qodo — "Best AI Code Review Tools of 2026," 2026. https://www.qodo.ai/blog/best-ai-code-review-tools-2026/
    - Context and enterprise scale in AI code review
    - Tool integration with CI/CD pipelines for automated review
    - Comparison of review-only tools vs. generate-and-review integrated platforms
