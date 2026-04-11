## Overview

Verification-driven development is the practice of structuring AI coding sessions so that Claude can verify its own output rather than requiring a human to catch every error. It is identified by Anthropic as "the single highest-leverage thing you can do" in a Claude Code session,[^1] and confirmed by Boris Cherny — who built Claude Code — as his number one practical tip: "Give Claude a way to verify its work. If Claude has that feedback loop, it will 2–3x the quality of the final result."[^2]

The principle is not new. Software engineers have understood for decades that fast, automated feedback loops produce better code than slow, manual ones. What is new is the degree to which it applies to AI-generated code specifically: an AI without a verification mechanism produces outputs that look correct but fail on edge cases, introduce subtle bugs, and drift from the intended specification. An AI with a tight verification loop — test run, build pass, screenshot comparison, linter check — self-corrects before the human ever sees the output.

This memo covers why AI needs external verification more than human engineers do, the TDAD academic framework for test-driven agentic development, practical test-first workflows, CI/CD as the terminal verification layer, and how to build verification infrastructure for code areas that currently lack test coverage.

---

## Section 1: Why AI Needs External Verification

**Description:** Human engineers who write incorrect code usually know something is wrong: their mental model of the system gave them expectations, the code violated those expectations, and cognitive dissonance signals the error. AI does not have a mental model; it has pattern-completion. If the pattern-completion produces plausible-looking code that is incorrect, there is no internal signal that something is wrong. Without external verification, incorrect AI code is indistinguishable from correct AI code at the generation stage.[^3]

This is not a theoretical concern. Anthropic's own documentation notes that without verification criteria, Claude "might produce something that looks right but actually doesn't work."[^1] The practical consequence is that the engineer becomes the only feedback loop — and every mistake requires human attention. At scale, this is not a sustainable review model. Issue 5 (Review Theater) documents the downstream consequence: reviewers become rubber stamps because the review burden per-PR has grown faster than review capacity.

The solution is to move verification upstream: into the session itself, before the human ever sees the output. When Claude runs tests and sees them pass, it has evidence of correctness. When it runs a linter and sees no errors, it has evidence of conformance. When it takes a screenshot and compares it to a design, it has evidence of visual correctness. Each verification mechanism Claude has access to is a feedback loop that reduces the error rate of its final output — and reduces the review burden that lands on the human.[^1]

The METR productivity study found a 19% slowdown from AI tool use on complex tasks — a result that correlates with insufficient verification infrastructure. Tasks where AI tools helped the most were those with fast, automated feedback loops; tasks where AI tools hurt the most were those where verification required human judgment at each step.[^4]

**Recommended Practice:**
- Treat verification infrastructure as a prerequisite for AI-assisted development, not an afterthought. Before beginning an AI session on a module with poor test coverage, invest time in adding at minimum a smoke test or integration test that Claude can run.[^1]
- Make verification part of every AI task prompt: "Implement X and run the test suite. Fix any failures before reporting completion." The difference in output quality between "implement X" and "implement X and verify it passes tests" is material.[^1]
- Use the Chrome DevTools MCP for UI verification: have Claude take a screenshot of its output and compare it against the design or expected state before the engineer reviews it.[^2]
- Track verification coverage as a team metric: which modules have verification Claude can run? Which don't? Prioritize adding verification infrastructure to modules with active AI-assisted development.[^3]

---

## Section 2: The TDAD Framework

**Description:** TDAD (Test-Driven Agentic Development) is an academic framework, published in a March 2026 paper, that formalizes the relationship between test suites and AI coding agents. The core insight is that AI agents do not need to be told how to do TDD; they need to be told which tests to check.[^5]

TDAD performs pre-change impact analysis: before an agent commits a patch, it builds a dependency map between source code and tests, identifying which tests are relevant to the changed code. The agent verifies these specific tests (not the entire suite, which may be prohibitively slow) and self-corrects if they fail. This delivers targeted, fast verification rather than the blunt-instrument approach of running every test on every change.[^5]

The empirical results are significant: TDAD reduced regressions from 6.08% to 1.82% — a 70% regression reduction — compared to baseline AI coding agent performance.[^5] This represents the quality ceiling available when verification is well-designed; the alternative is a 6% regression rate that accumulates across every AI-generated PR.

A counterintuitive finding from the TDAD research is important for teams designing verification workflows: adding TDD instructions (telling the agent to do test-driven development) without targeted test context *worsened* outcomes, increasing regressions to 9.94% — worse than no intervention. Prescribing a process (TDD) is less effective than providing context (relevant tests). The implication for practice is clear: give Claude specific tests to verify against, not general instructions to "write tests first."[^5]

**Recommended Practice:**
- Point Claude to specific test files relevant to the changes being made, not just the full test suite. "Run the tests in `auth/tests/` that are relevant to token refresh" is more effective than "run all tests."[^5]
- Configure the CLAUDE.md to describe which test commands to use for which module areas, so Claude applies the right verification automatically rather than guessing.[^1]
- Review the TDAD framework's impact analysis approach as a model for CI gate design: rather than running the entire test suite on every AI-generated PR, build a dependency-based gate that runs the tests relevant to changed files.[^5]
- Do not substitute TDD process instructions for test context. "Verify against tests/auth_test.go and tests/session_test.go" is more effective than "follow TDD practices."[^5]

---

## Section 3: TDD with Agentic AI

**Description:** The mainstream narrative is that AI-assisted development has disrupted TDD: when code is generated in seconds, the discipline of writing tests first seems to slow things down rather than speed them up. In practice, the relationship is the inverse — AI tools are most effective precisely when tests already exist, because the tests provide the verification mechanism that allows the AI to self-correct.[^6]

The practical TDD pattern for agentic AI sessions has three steps: write tests first (human or AI), hand implementation to Claude with the tests as verification targets, iterate until all tests pass.[^6] This gives Claude a concrete success criterion at every step — not "did I write plausible-looking code?" but "do these tests pass?" The quality difference between these two verification modes is material.

Steve Kinney's documented TDD workflow with Claude Code establishes this explicitly: tests as the "target" for AI implementation, rather than an afterthought. The test suite is the specification made executable — it describes the system's expected behavior precisely enough for Claude to evaluate whether its implementation is correct.[^7]

For teams with uneven test coverage, a pragmatic hybrid is available: write tests before handing any AI session its implementation task, even if tests don't already exist for the module. The overhead of writing three tests that define the expected behavior of an endpoint is small relative to the review and debugging time saved when Claude's implementation is verified against those tests.[^6]

**Recommended Practice:**
- Before beginning an AI implementation session, write tests that define the expected behavior. Even minimal tests — happy path, one edge case, one error case — give Claude a verification target.[^6]
- Use the pattern: "Write failing tests for X, then implement X until the tests pass, then run the full test suite to check for regressions." This is the complete TDD loop for AI-assisted development.[^7]
- For QA involvement: integrate QA early in the AI session workflow, not at the end. QA-written acceptance tests, run by Claude during implementation, move quality earlier in the process rather than later.[^3]
- Treat test coverage as a prerequisite gate for AI-primary stories in sprint planning: if a story involves modules with no test coverage, add test writing as a prerequisite task before AI implementation begins.[^6]

---

## Section 4: CI/CD as the Terminal Verification Layer

**Description:** CI/CD pipelines are the terminal verification layer — the last automated check before code reaches human review or production. For AI-generated code, configuring CI/CD as a first-class verification step is the safety net that catches what session-level verification missed.[^8]

Anthropic's documentation on hooks supports integrating CI/CD at the session level as well: hooks run automatically at defined points in Claude's workflow, allowing linters, type checkers, and tests to run automatically after each file edit rather than only at commit time. This creates a session-level feedback loop that catches errors before they accumulate into a correction spiral.[^1]

The safer CI pattern for agentic code review (documented by Roman Fedytskyi, March 2026) establishes a four-layer architecture: scope isolation (agents start with read-only access), context validation (semantic retrieval finds relevant files), validation stage (agent suggestions pass engineering and policy gates before application), and observability (full audit trails of prompts, tool calls, and validation results).[^9] For AI-generated code specifically, this pattern addresses the risk that an AI session makes changes that are locally valid but violate system-level security or operational constraints.

SAST scanning should be integrated into CI for AI-generated code, per Issue 4. The connection to verification-driven development is direct: SAST is a verification mechanism Claude can run against its own output during a session ("Run Snyk Code against the changes I've made and address any findings"), not just a post-session check.[^3]

**Recommended Practice:**
- Configure PostToolUse hooks to run linters and type checkers after each file edit. This creates a tight feedback loop within the session, not just at commit time.[^1]
- Add SAST to the list of verification tools Claude should run before considering a task complete: "Implement X, run tests, run the linter, and run Snyk against the changes."[^3]
- Use `claude -p "lint the changes vs. main and report issues"` in CI pipelines as an automated review step. This catches AI-specific patterns that standard linters miss.[^1]
- Separate the CI verification stages: test correctness (does it work?), security correctness (is it safe?), and architectural correctness (does it fit the system?). Each is a different verification mechanism requiring different tools.[^9]

---

## Section 5: Verification Without Tests

**Description:** Not all code areas have test coverage, and not all output types are testable by unit tests. Verification-driven development must address the modules where traditional testing is absent or impractical — not by abandoning verification, but by substituting other mechanisms.[^10]

For UI code without tests, visual verification is available: screenshot comparison tools allow Claude to compare its output against a reference design or expected state. The Chrome extension integration enables Claude to navigate to the running app, take a screenshot, and evaluate it against the design spec, iterating until the visual output matches.[^2]

For infrastructure code, linters, type checkers, and dry-run modes provide verification without test suites. Terraform `plan` output, Docker build success, and cloud provider validation modes all give Claude concrete pass/fail signals to verify against.[^1]

For data access code, query plan analysis, sample data checks, and row count validation provide lightweight verification without full integration test suites. "Run this query against the dev database and verify the result set is consistent with these three expected records" gives Claude a signal even in the absence of a test framework.[^10]

The common principle: every domain has a verification mechanism available. The work is identifying what that mechanism is for each area of the codebase and making it consistently available to AI sessions. This is a one-time infrastructure investment with compounding returns — it applies to every future AI session in that module.[^3]

**Recommended Practice:**
- Document the verification mechanisms for each codebase module in `CLAUDE.md`: which modules have test suites Claude should run, which have visual verification available, which have linters or dry-run modes.[^1]
- For UI-heavy modules, set up the Chrome extension and configure it as part of the verification step for any frontend AI session.[^2]
- For modules without test coverage, adopt the practice of writing minimal verification tests before beginning AI implementation sessions, even if a full test suite is not the goal.[^6]
- Create a "verification debt" list alongside the codebase: modules where no AI verification mechanism currently exists. Prioritize adding verification infrastructure to these modules as a team investment before scheduling AI-primary work in them.[^3]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Verification Prerequisites | Add "implement X and run tests" to all AI task prompts | All engineers |
| TDAD Implementation | Point Claude to specific relevant tests, not full suites; not TDD process instructions | All engineers |
| TDD Integration | Require test writing before AI implementation for AI-primary stories | QA + engineers |
| CI/CD Integration | Configure PostToolUse hooks for linters; add SAST to session verification | Backend lead |
| Non-Test Verification | Document verification mechanisms per module in CLAUDE.md | Architect |

---

[^1]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    - Verification as "the single highest-leverage thing you can do": why AI without verification produces plausible but incorrect output
    - PostToolUse hooks for session-level linting and type checking
    - Specific verification prompt examples: before/after comparison prompts, test-run-fix loop

[^2]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    - "#1 most important tip": 2–3x quality improvement from giving Claude a verification feedback loop
    - Chrome extension integration for visual UI verification within sessions
    - Verification-first workflow design: verification criteria before implementation, not after

[^3]: BitDive — "QA in AI-Assisted Development: Safety Through Deterministic Verification," 2026. https://bitdive.io/blog/quality-assurance-ai-assisted-software-development/
    - Deterministic verification as the counterweight to AI probabilistic generation
    - Verification mechanism taxonomy: tests, linters, SAST, visual checks, dry-run modes
    - Integration of QA earlier in the AI session workflow rather than at the end

[^4]: METR — "Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity," July 2025. https://arxiv.org/abs/2507.09089
    - 19% productivity slowdown on complex, high-context tasks without fast feedback loops
    - Fast feedback loop as the primary variable distinguishing productive from unproductive AI task types
    - Implication: verification infrastructure determines whether AI helps or hurts per task type

[^5]: TDAD Research — "TDAD: Test-Driven Agentic Development — Reducing Code Regressions in AI Coding Agents via Graph-Based Impact Analysis," arXiv, March 2026. https://arxiv.org/abs/2603.17973v2
    - 70% regression reduction: from 6.08% to 1.82% with TDAD vs. baseline AI agent performance
    - Counterintuitive finding: TDD process instructions without targeted test context increased regressions to 9.94% (worse than no intervention)
    - Core insight: "agents need to be told which tests to check, not how to do TDD"

[^6]: Coding Is Like Cooking — "Test-Driven Development with Agentic AI," March 2026. https://coding-is-like-cooking.info/2026/03/test-driven-development-with-agentic-ai/
    - TDD's evolving role: AI performance is highest precisely when tests already exist
    - Practical TDD pattern for AI sessions: write tests first → hand to Claude → iterate until passing
    - The "thinking stage" problem: TDD preserves the design thinking that AI-first generation skips

[^7]: Steve Kinney — "Test-Driven Development with Claude Code," AI Development Course, 2026. https://stevekinney.com/courses/ai-development/test-driven-development-with-claude
    - Tests as the "target" for AI implementation: executable specification vs. written specification
    - Complete TDD loop: failing tests → implementation → passing tests → regression check
    - Practical prompts for the TDD workflow with Claude Code

[^8]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
    - Four-layer CI architecture for AI-generated code: scope, context, validation, observability
    - Staged writes pattern: patches generated as files before application, human approval gate
    - Path restrictions and secret stripping in AI-accessible CI environments

[^9]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    - Claude as unix-style utility: `claude -p "lint changes vs. main"` in build scripts and CI
    - PostToolUse hook configuration for automatic post-edit verification
    - Headless mode for CI integration: non-interactive verification runs

[^10]: The Neuron — "Test-Driven Development for AI Coding: Beginner's Guide 2026," 2026. https://www.theneuron.ai/explainer-articles/test-driven-development-ai-coding-guide/
    - TDD for AI coding: practical guide for teams new to the integration
    - Non-test verification mechanisms: visual checks, dry-run modes, query validation
    - Building verification infrastructure before beginning AI-primary development

[^11]: Vibe Sparking AI — "VSDD: When Your AI Writes Code, Who Checks Its Homework?" March 2026. https://www.vibesparking.com/en/blog/ai/vibe-coding/2026-03-03-verified-spec-driven-development/
    - Verified Spec-Driven Development (VSDD): combining specification with adversarial verification
    - Hyper-critical reviewer pattern: verification continues until a reviewer is "forced to invent flaws because real ones no longer exist"
    - Integration of verification and specification as a unified workflow discipline
