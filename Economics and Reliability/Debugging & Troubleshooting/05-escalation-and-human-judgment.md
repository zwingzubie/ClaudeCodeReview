## Escalation and Human Judgment

**Related to:** [Debugging & Troubleshooting Overview](00-overview.md) — Debugging Area 5 · [Workflows: Task Decomposition](../Workflows/02-task-decomposition.md) · [Governance: Escalation Procedures](../Governance/04-escalation-procedures.md) · [Issues: Skill Atrophy](../Issues/06-skill-atrophy.md) · [Learning: Skill Maintenance](../Learning/02-skill-maintenance.md)

---

## Overview

Escalation — recognizing when AI iteration has reached its limit and human judgment is required — is the AI-assisted development skill that receives the least attention and causes the most damage when absent. Teams discuss how to prompt better, how to structure sessions, and how to verify output; they rarely discuss how to know when to stop using AI for a given problem and bring in human expertise instead. The result is a failure mode where engineers continue iterating with AI on a problem that AI cannot solve, generating cost and consuming time while making no progress toward resolution.[^1]

The case for explicit escalation criteria is not that AI is insufficient for most engineering work — it is that AI is insufficient for a specific, identifiable subset of engineering tasks, and the cost of misidentifying a task as AI-solvable when it is not is a function of how long escalation is delayed. An hour of unsuccessful AI iteration before escalation represents both the direct cost of the session and the indirect cost of a one-hour delay in getting human expertise applied to the problem. For critical-path bugs and security issues, that delay has disproportionate impact.

Escalation is not a failure of AI adoption. It is accurate calibration of AI's current capabilities against the team's task distribution. A team that escalates appropriately uses AI for the 80% of tasks it handles well and reserves human effort for the 20% that require judgment, organizational context, or domain expertise that AI cannot provide — which is the optimal allocation of both resources.[^3]

---

## Section 1: When AI Iteration Has Reached Its Limit

**Description:** The primary escalation signal is failure to converge despite correct conditions: the task is well-specified, the model tier is appropriate, the context is complete and accurate, and the session has been managed correctly — yet after three or more sessions the output is not approaching the target. When these conditions are all met and the task is still failing, the problem is the task itself, not the session conditions. That is the signal that the task is outside AI's current reliable range.[^4]

Secondary escalation signals supplement the primary convergence test: the task requires organizational knowledge that is not documentable as session context (why a particular architectural decision was made for a business reason not recorded in the codebase, what a stakeholder's implicit requirements are, what the regulatory interpretation of a specific compliance requirement should be); the task requires live system access, operational judgment, or real-time troubleshooting that the AI cannot do; or the task involves business trade-offs where the correct answer depends on priorities that the AI cannot access.[^5]

**Recommended Practice:**
- Define explicit escalation criteria in the AI usage policy and share them with the engineering team: "Escalate to human judgment when (a) three sessions at Opus tier with complete specification and context have not produced correct output, (b) the task requires organizational context that cannot be documented, (c) the task requires live system access or real-time operational judgment, or (d) the task's correct answer depends on business trade-off priorities not codified in CLAUDE.md."[^3]
- Apply the three-session rule consistently, not as a last resort after extended iteration. Engineers who apply the rule consistently reach escalation faster than those who override it with "just one more session" reasoning. The three-session rule's value is in converting an open-ended iteration into a decision: three sessions attempts, then escalate.[^4]
- Do not treat escalation criteria as a checklist to game. The purpose of explicit criteria is to accelerate the escalation decision when it is correct, not to create a formal process that must be completed before a decision the engineer already knows is right. If the engineer recognizes before session three that the task requires organizational knowledge AI cannot have, escalate before session three.
- Review escalation decisions at the monthly AI practice meeting: were they made at the right time? Were there tasks where earlier escalation would have been better? Were there tasks where escalation was avoided and AI eventually succeeded — evidence that the escalation criteria should be calibrated differently? This review keeps the escalation criteria accurate rather than static.

---

## Section 2: Escalation as a Workflow Step

**Description:** The cultural obstacle to timely escalation is the perception that escalation means AI has "failed" or that the engineer using AI has made a mistake. This framing is counterproductive: it creates pressure to continue AI iteration past the point of diminishing returns in order to avoid the appearance of failure, when the correct outcome for the task and the team is to bring human expertise to bear at the right time.[^1]

Escalation should be documented in team workflow as a normal, expected step — not an exceptional event. The Governance section's escalation procedures document provides the organizational context for this normalization; the debugging section's escalation criteria provide the technical triggers. Together, they create the conditions for engineers to escalate without cultural friction.[^5]

**Recommended Practice:**
- In the team's sprint workflow, include escalation as an explicit workflow status alongside "in progress" and "complete": a task can be "escalated" with documentation of when it was escalated, why, and to whom. This makes escalation a visible, trackable workflow step rather than an invisible event that the engineer handles informally.[^3]
- When escalating, document the session history for the person receiving the escalation: what was attempted, what outputs were produced, what specifically failed, and what information about the problem was surfaced by the AI sessions. This handoff documentation makes the human's starting point better than if no AI work had been done — the AI sessions have value even when they did not produce a solution.[^4]
- Define escalation recipients for common task categories: for architectural decisions, the architect; for production incidents, the on-call engineer and architect; for security issues, the architect and CTO; for business logic requiring stakeholder input, the relevant product manager. Engineers who know who to escalate to are faster to escalate than engineers who have to figure it out each time.[^5]
- Reward timely escalation in team culture. A public acknowledgment — "good call escalating this, the session logs showed the problem required organizational context AI couldn't have" — reinforces the behavior. Cultural norms are shaped by visible responses to specific actions; normalizing escalation requires visible positive responses when it is done well.

---

## Section 3: Human-AI Collaboration in Escalated Tasks

**Description:** Escalation does not mean abandoning AI assistance for the task — it means bringing human judgment into the loop to complement what AI can and cannot do. In many escalated tasks, the most effective resolution is a human-AI collaboration: the human provides the organizational context, business judgment, or expert domain knowledge that AI cannot access, while AI accelerates the implementation once the human has resolved the ambiguity that prevented AI from succeeding alone.[^6]

This collaboration model changes the nature of escalation from "stop using AI" to "use AI differently." The human's role in the escalated task is to provide the inputs that AI needs — a clear answer to the business trade-off, the organizational context for the architectural constraint, the expert diagnosis that the AI's sessions were approximating — and then to use AI to execute the implementation once the blocking problem is resolved.

**Recommended Practice:**
- When escalating, identify specifically what input the human needs to provide in order to unblock AI execution. This framing converts an open-ended escalation into a targeted request: "I need the architect to decide whether we should use optimistic or pessimistic locking here — the AI correctly identified the tradeoffs but can't make the business judgment. Once we have a decision, I'll implement it with AI assistance."[^6]
- After a human provides the resolving input, document it in CLAUDE.md or the task spec as an explicit fact rather than relying on the engineer's memory. The human's judgment call should be encoded as context for future sessions touching the same area, converting the escalation into a long-term governance improvement.[^4]
- For escalated tasks where the human's resolution is a decision that will apply to future tasks of the same type, encode the decision pattern in CLAUDE.md: "When [condition], use [approach]. This was decided by [owner] because [reason]." This transforms a one-time human judgment call into a standing AI instruction that prevents future escalation for the same decision.[^3]
- Track the ratio of escalation-resolved-by-human to escalation-resolved-by-updated-context: if most escalations are resolved by a human making a decision rather than by adding context AI could have used, the team's task taxonomy is correctly identifying AI's limits. If most escalations could have been resolved by better context injection, the escalation is happening before it should — the correct intervention is context improvement, not escalation.

---

## Section 4: Building Domain Expertise Alongside AI Use

**Description:** The escalation requirement for organizational knowledge and expert domain judgment is not static — it decreases as CLAUDE.md accumulates organizational context, as the prompt library grows with proven patterns, and as the team's task taxonomy becomes more precisely calibrated to actual AI capabilities. Over time, tasks that required escalation may become AI-solvable because the blocking organizational context has been encoded. Tracking which escalations become encodable is a leading indicator of the team's AI governance maturation.[^1]

There is a complementary risk: engineers who exclusively use AI for complex work may not maintain the domain expertise required to escalate correctly. Knowing when to escalate requires understanding what the correct answer should look like — which requires domain knowledge that must be actively maintained through practice, not just through observation of AI output. The Learning section addresses this explicitly; the debugging section's connection is that effective escalation is a domain-expertise-dependent skill.[^7]

**Recommended Practice:**
- Designate specific task types as "human-first" for each engineer as part of their skill maintenance plan: tasks that the engineer will continue to complete without AI assistance to preserve the domain expertise required for escalation and oversight in those areas. Engineers who do not occasionally solve complex problems without AI assistance gradually lose the ability to evaluate whether AI is solving them correctly.[^7]
- After encoding an organizational decision in CLAUDE.md (converting a prior escalation into an AI instruction), test whether similar tasks can now be completed without escalation. If so, update the task taxonomy to reflect the new capability boundary. Task taxonomies should be living documents that evolve as CLAUDE.md context grows.[^3]
- Review escalation logs quarterly for patterns: which task types escalate repeatedly, which escalations resulted in encodable context vs. non-encodable human judgment, and whether the escalation rate for specific task types is changing over time. A declining escalation rate for a task type indicates that CLAUDE.md context for that type is maturing; a stable or rising rate indicates that the blocking factor is organizational judgment that may not be fully encodable.[^4]
- Share escalation logs across the team so that all engineers can see which task types other engineers are escalating. This shared visibility prevents the situation where multiple engineers independently discover that a task type requires escalation, each spending three sessions learning what the others already know. Shared escalation knowledge is the team-level version of the individual pre-reset diagnostic.

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Escalation Criteria | Document explicit escalation criteria in AI usage policy; share with team | Architect |
| Escalation as Workflow | Add escalation status to sprint tracking; define escalation recipients by category | Architect |
| Human-AI Collaboration | Train engineers on the "unblock then execute" collaboration model | Architect |
| Domain Expertise Maintenance | Assign human-first task types to each engineer for skill preservation | Architect |

---

[^1]: Anthropic — "2026 Agentic Coding Trends Report," Anthropic, 2026. https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf
 Escalation as accurate capability calibration rather than AI failure; the cost of delayed escalation on critical-path tasks; escalation normalization in team culture; the 80/20 AI-human task allocation model.

[^3]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
 Explicit escalation criteria in AI usage policy; escalation as a normal workflow step; encoding resolved escalations in CLAUDE.md; task taxonomy evolution as CLAUDE.md context grows.

[^4]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
 The three-session rule in practice; escalation handoff documentation; tracking escalation-to-encoding conversion rate; shared escalation logs as team knowledge infrastructure.

[^5]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
 Escalation recipients by task category; escalation workflow integration with sprint tracking; escalation documentation for handoff quality; secondary escalation signals for organizational context gaps.

[^6]: CIO — "How Agentic AI Will Reshape Engineering Workflows in 2026," April 2026. https://www.cio.com/article/4134741/how-agentic-ai-will-reshape-engineering-workflows-in-2026.html
 Human-AI collaboration in escalated tasks; the "unblock then execute" model; encoding human judgment decisions as standing AI instructions; escalation as a governance improvement trigger.

[^7]: Issues — "Skill Atrophy," Issues/06-skill-atrophy.md
 Human-first task designation for domain expertise preservation; the relationship between escalation quality and maintained domain expertise; skill maintenance as a prerequisite for effective AI oversight.
