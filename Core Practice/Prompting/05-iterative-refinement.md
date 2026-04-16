## Iterative Refinement: Getting to the Right Output Through Structured Loops

**Related to:** [Prompting Overview](00-overview.md) — Pattern 4 · [Workflows: Verification-Driven Development](../Workflows/05-verification-driven-development.md)[^a] · [Metrics: Rework Rate](../Metrics/03-rework-rate.md)[^b] · [Workflows: Session Hygiene](../Workflows/04-session-hygiene.md)[^c]

---

## Overview

Single-shot prompting — the attempt to specify a task completely enough that the first output is acceptable — fails systematically for complex tasks. The failure is not a model deficiency; it is structural. Complex tasks contain specification gaps that are invisible until the first output reveals them. The first output is not a failure when it misses — it is a diagnostic. It reveals what the specification was ambiguous about, what constraints were implicit, and what the engineer's mental model assumed Claude would infer. Iterative refinement is not a workaround for imprecise prompting; it is the correct process for complex tasks where complete upfront specification is not achievable at the cost of writing the first prompt.[^1]

The key discipline in iterative refinement is treating each output as signal, not as a final answer to accept or reject. The question after every output is not "is this good enough to ship?" but "what does this reveal about what I need to specify more precisely?" Engineers who ask the second question converge faster than those who ask the first, because they use each iteration to reduce the specification gap rather than to nudge the existing output toward a result they cannot quite define.[^2]

---

## Section 1: Why Iterative Refinement Matters

**Description:** The cost of trying to specify everything upfront is that complex tasks have specification gaps that only become visible when implementation begins. An engineer writing a prompt for a feature implementation may not know whether the authentication pattern they have in mind is compatible with the existing session management until Claude attempts to implement it. The attempt surfaces the incompatibility. If the process requires complete upfront specification, the engineer must discover and resolve all such conflicts before writing the first prompt — a form of pre-implementation that is often slower than letting the first output reveal the conflicts and resolving them iteratively.[^3]

Single-shot prompting also carries a specific quality risk: it tends to produce confident, plausible outputs that are wrong in non-obvious ways. When Claude does not know enough to implement correctly, it implements based on defaults — and defaults produce code that looks correct and fails subtly. Iterative refinement catches these failures earlier, because each round's output is evaluated against progressively sharper specification. The first output may be wrong in obvious ways; by the third or fourth iteration, the only remaining issues are the non-obvious ones that require the closest engineering attention.[^4]

**Recommended Practice:**
- For any task estimated to take more than thirty minutes of engineering time, plan for at least two iteration rounds before the output is implementation-ready. The planning cost of expecting iteration is lower than the debugging cost of treating a single-shot output as correct when it is not.[^1]
- Distinguish between tasks where single-shot is appropriate (bug fixes with a clear definition of correct, changes with high codebase precedent, simple additions following an established pattern) and tasks where iteration is structurally necessary (new feature implementations, architectural changes, anything requiring judgment about tradeoffs). Apply the processes differently.[^5]
- Track which task types in your codebase reliably require iteration to reach acceptable quality. Patterns that emerged from iterative sessions should inform the prompt library and eventually the CLAUDE.md specification, so future sessions start closer to the converged result from the start.[^2]
- When time pressure pushes toward single-shot prompting for a task that structurally requires iteration, use the saved time for a faster review of the output — not for skipping review. The output from a single-shot complex task has a higher defect rate and requires correspondingly more careful engineer review to catch what the iteration would have surfaced.[^3]

---

## Section 2: The Clarification Loop

**Description:** The clarification loop is an inversion of the usual prompt-response pattern: instead of the engineer specifying everything before Claude responds, the engineer asks Claude to surface what it needs before implementing. "Before you implement this, list the questions whose answers would most affect how you implement it." This prompt converts Claude from a default-filling machine into a specification-gap detector. The questions it surfaces are the places where the engineer's mental model was incomplete or ambiguous; the engineer's answers to those questions constitute the missing specification.[^6]

The clarification loop works best when constrained to high-value questions. An unconstrained clarification request ("ask me anything you need") produces exhaustive question lists that are tedious to answer and contain many questions about things Claude should determine from the codebase rather than asking. A constrained request ("list the three questions whose answers would most change your implementation approach") produces focused, high-signal questions that are fast to answer and concentrate the specification work where it matters most.[^7]

**Recommended Practice:**
- Use the constrained clarification prompt before any non-trivial implementation: "Before implementing, list the three questions whose answers would most affect your approach. Do not implement until I have answered them." The constraint on three questions forces Claude to prioritize rather than enumerate, producing a more useful clarification exchange.[^6]
- When a clarification question reveals a genuine ambiguity in the requirements (not in Claude's knowledge of the codebase), the answer belongs in spec.md, not just in the session. Capture it before continuing so future sessions have the resolved specification.[^5]
- Distinguish between clarification questions about facts (Claude should read the relevant files) and clarification questions about intent (only the engineer can answer). Answering factual questions that Claude should resolve by reading the codebase trains a pattern of dependency; deflecting them — "read `@src/auth/` and determine this yourself" — is better practice.[^8]
- After a clarification loop produces a clean implementation, review the questions that were asked. Recurring clarification questions across multiple sessions for the same task type are candidates for CLAUDE.md additions — they represent specification that should be globally available rather than re-established per session.[^2]

---

## Section 3: Output Evaluation and Correction Prompts

**Description:** The correction prompt is the most common iterative refinement action, and the most common way it goes wrong is by being too broad. "This isn't right, try again" is not a correction prompt — it is a new shot at the same underspecified task. An effective correction prompt identifies specifically what is wrong, explains why it is wrong (not just that it is wrong), and preserves what is right so Claude does not change it. The precision of the correction determines the precision of the correction's output; vague corrections produce vague improvements.[^9]

The restart vs. correction decision is the second most important judgment in iterative refinement. Correction is appropriate when the output is directionally right and specifically wrong — the right approach with execution errors, the right structure with incorrect details. Restart (using `/clear`) is appropriate when the output is directionally wrong — using the wrong pattern entirely, misunderstanding the fundamental task, accumulating corrections that contradict each other. Persisting with corrections when the direction is wrong produces an output that is correct at the surface while remaining wrong at the architecture.[^10]

**Recommended Practice:**
- Structure correction prompts with three components: what is wrong (specific), why it is wrong (the violated constraint or missed requirement), and what must not change (preservation). "The error handling in `processPayment` is wrong — it swallows the `PaymentDeclinedError` instead of surfacing it to the caller, which violates our error propagation convention. Fix only the error handling; do not change the validation logic or the return types."[^1]
- Use the preserve-what-is-good pattern explicitly: "Keep everything above line 40. Rework only the `handleRetry` function." This prevents correction loops from drifting the implementation as a whole rather than fixing the specific defect identified.[^3]
- Apply the direction test before the third correction on the same output: if three corrections have not converged, diagnose whether the output is directionally wrong. Restart with `/clear` and a prompt that incorporates what the correction rounds revealed about the missing specification. Three rounds of diverging corrections are evidence of a direction problem, not an execution problem.[^10]
- Log the correction prompt alongside the output it corrected during a development session. After the session, review which corrections were most common. Recurring corrections are specification gaps that should be closed in CLAUDE.md or the prompt template for the task type, not managed as correction overhead every session.[^4]

---

## Section 4: Convergence Patterns

**Description:** Iterative refinement converges when successive outputs change less in each round. The practical signal for convergence is that a correction prompt produces an output where only the specific targeted issue changed — nothing else shifted. Divergence, the dangerous opposite, occurs when each correction produces new unexpected changes alongside the targeted fix: code that was correct in round two is different in round three, not because it was corrected but because the correction round reorganized the implementation in ways that were not requested. Divergence is a signal that the implementation approach needs to be reset, not refined further.[^11]

The "good enough" determination is not a quality judgment alone — it is a quality-versus-cost judgment. The question is not whether further iteration would improve the output but whether the improvement from further iteration is worth the cost of producing it. For code that will be heavily reviewed by a senior engineer before merge, a third iteration that catches a pattern inconsistency is worth less than for code that will receive minimal review. Knowing when iteration is delivering diminishing returns requires calibrating the cost of remaining defects against the cost of additional iteration rounds.

**Recommended Practice:**
- Define a convergence criterion before beginning iteration on a task: what specific quality bar must the output meet for iteration to stop? "All existing tests pass, the new endpoint returns the expected response structure in manual testing, and no new patterns inconsistent with the codebase are introduced." Undefined convergence criteria produce indefinite refinement.[^9]
- Use the delta test to assess convergence: after each iteration, identify what changed. If the delta is targeted (only what was corrected changed), convergence is proceeding. If the delta is scattered (many things changed in response to a specific correction), convergence is not occurring — the implementation is still unstable.[^11]
- Hand off to human review when the remaining issues are judgment calls rather than specification gaps. Issues that require human architectural judgment are not solvable by further iteration; they are solvable by human decision-making followed by a targeted implementation prompt. Iterating past this point is not convergence — it is cycling.[^5]
- Track iteration counts per task type across sessions. Consistently high iteration counts for a task type indicate missing specification that should be in the prompt template; consistently low iteration counts indicate task types that are good candidates for more automated handling.[^2]

---

## Section 5: Building Iteration Templates for Common Task Types

**Description:** An iteration template is a pre-designed sequence of prompts for a recurring task type: a starter prompt, a clarification prompt, a standard correction pattern, and a convergence verification. For task types the team executes frequently — feature implementation from a spec, refactoring to a pattern, generating tests for an existing module — iteration templates eliminate the overhead of designing the refinement process from scratch each time. The first use of a task type in a new session starts at the clarification loop rather than at a blank prompt.[^13]

Iteration templates accumulate from real sessions. Every time a session finds an effective clarification question for a task type, that question belongs in the template. Every time a specific correction prompt form produces clean, targeted results, that form belongs in the template. The template library is a living artifact — it begins sparse and gains density as teams observe what works. A template library that is three months old and maintained by a team that reflects on their sessions is more valuable than any generic prompt library.[^14]

**Recommended Practice:**
- Create iteration templates for the three to five task types that account for the majority of AI-assisted development work. For most teams, these are: new feature implementation, targeted refactoring, test generation for an existing module, bug investigation and fix, and code review (writer/reviewer pattern). Start with templates for the two most frequent.[^13]
- Structure each template as a sequence: (1) session-setup prompt with task-type-appropriate context loading, (2) constrained clarification prompt, (3) implementation prompt incorporating clarification answers, (4) standard verification prompt, (5) correction prompt template. Engineers using the template fill in the task-specific details at each step.[^8]
- Store iteration templates in the shared prompt library (see 06-prompt-library.md) under a `templates/` subdirectory, with the task type as the filename. Templates are the highest-value prompt library content: they encode not just a single prompt but a complete interaction design for a task type.[^14]
- Retire templates when a task type is handled sufficiently well in CLAUDE.md and the prompt patterns have become automatic for the team. A template that every engineer already follows from memory is documentation overhead; a template for a task type where engineers still struggle is still active value.[^5]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Iteration Planning | Classify recurring task types as single-shot vs. iteration-required; document in onboarding | Architect |
| Clarification Loop | Add constrained three-question clarification prompt to command library as standard pre-implementation step | Individual engineers |
| Correction Prompt Structure | Adopt three-component correction prompt format (what, why, preserve); add to prompt style guide | Individual engineers |
| Convergence Criteria | Add explicit convergence criterion definition to task-start template in command library | Individual engineers |
| Iteration Templates | Build iteration templates for top three task types; store in prompt library under `templates/` | Architect |

---

[^1]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
 Iterative refinement as the correct process for complex tasks: why complete upfront specification fails structurally; using first outputs as diagnostic signal rather than final answers.

[^2]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
 Specification gap tracking across sessions; recurring clarification questions as CLAUDE.md candidates; iteration count as a task-type quality signal.

[^3]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
 Single-shot vs. iterative task classification; why complex tasks require iteration structurally; the cost of treating single-shot output from a complex task as review-ready.

[^4]: Fannar Steinn Aðalsteinsson et al. — "Rethinking Code Review Workflows with LLM Assistance: An Empirical Study," arXiv:2505.16339, May 22, 2025. https://arxiv.org/abs/2505.16339
 Single-shot output defect rates on complex tasks: empirical evidence that iteratively refined outputs have lower defect rates than single-shot outputs at equivalent specification levels.

[^5]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
 Task classification for single-shot vs. iteration; hand-off to human review when remaining issues require judgment; agentic session design for tasks with low iteration requirements.

[^6]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
 Clarification loop mechanics; pre-implementation question prompts; the Explore-Plan-Code workflow as the canonical iterative interaction pattern.

[^7]: Judy Shen and Alex Tamkin — "How Instruction Following Affects Context Use in Large Language Models," Anthropic / arXiv:2601.20245, January 2026. https://arxiv.org/abs/2601.20245
 Constrained clarification prompts: how question-count constraints improve the relevance and priority-ordering of clarification questions; unconstrained clarification request failure modes.

[^8]: Ravikanth Konda — "Patterns for Effective AI-Assisted Software Development," International Journal of AI in Business, Data and Cloud Management Systems, February 2026. https://ijaibdcms.org
 Iteration template design: structured prompt sequences for recurring task types; the five-step template structure for feature implementation; template libraries as team knowledge artifacts.

[^9]: Sreecharan Sankaranarayanan — "Towards Reliable AI Code Agents: A Framework for Evaluating Context Window Management," arXiv:2602.20206, February 2026. https://arxiv.org/abs/2602.20206
 Correction prompt precision: how the specificity of a correction prompt determines the precision of its output; the structural requirements of an effective correction vs. a re-try.

[^10]: METR — "We Are Changing Our Developer Productivity Experiment Design," METR Research, February 2026. https://www.metr.org/blog/2026-02-24-uplift-update/
 Restart vs. correction decision: the productivity cost of persisting with corrections when the direction is wrong; the direction test applied to three-round correction sequences.

[^11]: Kyros — "The Vibe Coding Crisis: How AI-Generated Technical Debt Is Costing Companies Millions," March 2026. https://usekyros.ai/blog/vibe-coding-crisis-ai-technical-debt
 Convergence vs. divergence patterns: the delta test for assessing convergence; how diverging corrections produce surface-correct but architecturally-wrong implementations.

[^13]: DEV Community — "AI Is Creating a New Kind of Tech Debt — And Nobody Is Talking About It," March 2026. https://dev.to/harsh2644/ai-is-creating-a-new-kind-of-tech-debt-and-nobody-is-talking-about-it-3pm6
 Iteration templates as technical debt mitigation: how pre-designed iteration sequences for recurring tasks prevent the ad-hoc prompting patterns that produce inconsistent-quality outputs.

[^14]: Lakera — "The Ultimate Guide to Prompt Engineering in 2026," Lakera Blog, 2026. https://www.lakera.ai/blog/prompt-engineering-guide
 Template library accumulation from real sessions: how to extract iteration templates from retrospective session review; template maturation patterns over time.

[^17]: Sabrina Ramonov — "CLAUDE CODE FULL COURSE," YouTube, February 17, 2025. https://www.youtube.com/watch?v=fYX6hHC9FhQ
 - Iteration planning: how to classify a task as single-shot vs. iteration-required at the start of a session and the different session designs for each
 - Preserve-what-is-good pattern: demonstration of correction prompts that explicitly scope the correction to avoid unintended changes to correct prior-round output
 - Iteration templates in the prompt library: how templates are stored, referenced, and updated based on new session learnings

[^a]: [Workflows: Verification-Driven Development](../Workflows/05-verification-driven-development.md) — verification-driven development structures the feedback loop that iterative refinement depends on; the ability to verify output is what makes structured refinement possible.
[^b]: [Metrics: Rework Rate](../Metrics/03-rework-rate.md) — iterative refinement within a session reduces post-merge rework; the metric captures whether refinement is happening at the session level or the post-merge level.
[^c]: [Workflows: Session Hygiene](../Workflows/04-session-hygiene.md) — session hygiene governs when to end versus continue a session; iterative refinement requires knowing when the session context has degraded to the point where refinement will not converge.