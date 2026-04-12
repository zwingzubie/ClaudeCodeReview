## Context Degradation in Long Sessions

**Related to:** [Debugging & Troubleshooting Overview](00-overview.md) — Debugging Area 3 · [Cost & Token Economics: Context Optimization](../Cost%20&%20Token%20Economics/03-context-optimization.md) · [Workflows: Session Hygiene](../Workflows/04-session-hygiene.md) · [Metrics: Session and Context Efficiency](../Metrics/00-overview.md)

---

## Overview

Context degradation is the gradual decline in Claude Code session quality as the context window fills with conversation history. It is distinct from session failure (where something structural goes wrong abruptly) and from task difficulty (where the problem itself is hard). Context degradation is a function of session length: given enough turns, every session will eventually experience it. The question is whether the team recognizes it early, manages it proactively with `/compact` or session resets, or allows it to accumulate until the session is producing output too degraded to use.[^1]

The degradation mechanism is straightforward. The model attends to context proportional to its relevance — when the context window is small and focused, the model's attention is concentrated on the most relevant material. As conversation history fills the context window, the proportion of the window occupied by directly relevant content decreases. The model continues to generate plausible output, but with diminishing fidelity to the specific constraints, decisions, and context established early in the session. The engineer experiences this as responses becoming less precise, constraints being forgotten, and the session requiring more correction cycles to achieve acceptable output — all symptoms that increase cost and reduce quality simultaneously.[^2]

---

## Section 1: Early Degradation Signals

**Description:** Context degradation has early warning signs that appear before quality declines to the point of generating obviously wrong output. Recognizing these early signals enables proactive management — a compact or session reset before the engineer has invested additional turns in degraded output. The early signals are subtle: they require active monitoring rather than passive observation.[^3]

The three primary early signals are: constraint drift (the model begins applying constraints established early in the session less consistently — using a prohibited pattern, generating code for a file declared out of scope, or ignoring an architectural principle from the CLAUDE.md); precision decline (responses become slightly less specific than they were earlier in the session — more generic examples, less precise variable names, more hedging language like "you might" rather than definitive recommendations); and history inaccuracy (when the model references earlier decisions in the conversation, it references them imprecisely or with details slightly different from what was actually established).[^1]

**Recommended Practice:**
- During sessions longer than 10 turns, periodically test the model's attention to established context: ask it to restate a constraint from CLAUDE.md, summarize the key decisions made in the session, or describe the scope boundary established in the session brief. Imprecise or incorrect responses to these test questions are a reliable early degradation signal.[^3]
- Track response length and hedging language as informal degradation proxies. A response to the same type of question that is 30% shorter and uses more conditional language ("might," "could," "generally") than an earlier response in the same session is exhibiting a precision decline consistent with early degradation.[^2]
- Set a turn-count soft limit for session types: for implementation sessions, treat 20 turns without a compact as a prompt to assess degradation signals. For exploratory sessions, treat 15 turns as the assessment prompt. These thresholds are not hard rules — they are reminders to actively evaluate whether the session is still functioning at full quality.[^4]
- When a degradation signal is detected, immediately apply `/compact` rather than continuing through more turns. The value of early detection is lost if it does not trigger prompt intervention. A compact at the first signal costs one turn; continuing for five more turns after the first signal generates five turns of degraded output before the inevitable compact.

---

## Section 2: Proactive Compact Strategy

**Description:** The `/compact` command replaces conversation history with a summary, resetting the context window to approximately the size of the summary plus the base context (CLAUDE.md, injected files). Used reactively — after quality has visibly declined — it restores quality but discards the turns that produced degraded output. Used proactively — at planned phase transitions before degradation appears — it maintains quality throughout the session and reduces total session cost by preventing the per-turn cost escalation that accompanies context growth.[^4]

The optimal compact timing is at natural session phase boundaries: after exploration is complete, after a design decision is finalized, after an implementation phase is complete, or after a debugging hypothesis is tested. At these boundaries, the content that will be compacted (the exploration/design/implementation turns) is fully complete and its key conclusions can be captured in the compact summary. The content that will be retained for the next phase (the compact summary plus injected files) is precisely the information the next phase needs.[^1]

**Recommended Practice:**
- Plan compact points before the session begins for sessions with multiple defined phases. A session with phases [explore current implementation → design refactoring approach → implement refactoring → write tests] should plan compacts after exploration, after design, and after implementation. Pre-planned compacts execute at better timing than reactive compacts because they are not triggered by degradation — they are triggered by phase completion.[^4]
- Before applying `/compact`, manually note any critical decisions, constraints, or specific values from the session that the compact summary might not capture. The compact summary is generated by the model and may omit specifics that the engineer knows will be important in the next phase. A brief pre-compact note ("Before compacting: note that we decided to use optimistic locking, not pessimistic, and the threshold is 500ms") prevents the information from being lost.[^3]
- Review the compact summary the model generates before proceeding. A summary that omits a key constraint or decision is a compact that will produce a degraded next phase even though the context window has been reset. Add any missing critical information explicitly to the first message of the post-compact phase.[^2]
- For sessions that have already experienced visible degradation, compact immediately rather than attempting additional correction turns. A session in visible degradation is not going to self-correct through iteration — the degraded context is contaminating each new turn. Compact, review the summary, supplement it with any critical information the summary missed, and continue from the reset state.

---

## Section 3: Session Reset vs. Compact

```mermaid
flowchart TD
    A{Degradation detected<br/>or turn-count limit hit} --> B{Is session output<br/>mostly correct so far?}
    B -- Yes --> C{Primary problem:<br/>context size?}
    C -- Yes --> D[/compact Focus on current phase]
    D --> E[Review compact summary]
    E --> F{Summary captures<br/>critical decisions?}
    F -- No --> G[Add missing info<br/>to first post-compact message]
    F -- Yes --> H[Continue session<br/>from reset context]
    G --> H

    B -- No --> I{Multiple turns going<br/>in wrong direction?}
    I -- Yes --> J[Extract useful fragments<br/>before closing]
    J --> K[Write revised session brief<br/>addressing root cause]
    K --> L[Fresh session reset]
    I -- No --> D

    C -- No --> M{Specification problem<br/>or wrong model tier?}
    M -- Yes --> J
    M -- No --> D
```

**Description:** Compact and session reset are both context management interventions, but they address different situations. Compact is appropriate when the session has made meaningful progress that should be preserved and the primary problem is context size. Session reset is appropriate when the session has gone significantly off course — accumulated incorrect state, gone in the wrong direction for multiple turns, or established incorrect assumptions that the compact summary would also include. Applying compact when reset is needed preserves the incorrect state in the summary; applying reset when compact would suffice discards progress unnecessarily.[^5]

The decision criterion is the quality of what the session has produced: if the session's output to this point is largely correct and useful, compact and continue; if the session's output has been going in the wrong direction and a significant proportion of what would be captured in the summary is incorrect, start fresh with a better specification. The question is not "how much have I invested" — sunk cost reasoning leads to continuing bad sessions past the optimal stopping point — but "what is the starting state I want for the next phase of this work?"[^1]

**Recommended Practice:**
- Use a reset when the session has produced substantially incorrect output across multiple turns — code going in the wrong architectural direction, tests written for the wrong behavior, or a debugging hypothesis that has been applied incorrectly for several turns. In these cases, the compact summary will capture incorrect conclusions that will contaminate the post-compact session.[^5]
- Before resetting, extract any useful information from the failed session: individual correct observations, useful error messages, confirmed hypotheses, or partial implementations that are correct even if the overall direction was wrong. These can be injected as starting context for the fresh session rather than being re-derived.[^3]
- When starting a fresh session after a reset, write a revised session brief that explicitly addresses the reason the prior session failed: if it failed because of an underspecified requirement, the new brief should specify that requirement explicitly; if it failed because of incorrect context injection, the new brief should inject the correct context. The reset session should not repeat the conditions that caused the prior session to fail.[^2]
- Track session resets in the session log alongside compacts. A high reset rate relative to compacts indicates that sessions are frequently going significantly wrong rather than gradually degrading — a pattern that warrants review of the team's session scoping and brief-writing practices rather than just their compact timing.

---

## Section 4: Long-Session Architecture for Complex Tasks

**Description:** Some tasks genuinely require more turns than a single well-managed session can accommodate without significant degradation — complex multi-file refactoring, extended architectural analysis, or step-by-step debugging of a non-obvious bug. For these tasks, the answer is not to attempt the task in a single session and manage degradation as it occurs. It is to architect the task as a multi-session workflow from the outset, with each session having a defined scope, a defined output, and a defined handoff to the next session.[^6]

Multi-session task architecture converts the context degradation problem from a mid-session emergency into a planned workflow. Each session starts fresh with clean context, has a clear scope and termination criterion, and produces a documented output that becomes the starting context for the next session. The sessions are shorter, higher quality, and less expensive than a single long session attempting the same work — because each session's context remains focused and relevant throughout its duration.[^2]

**Recommended Practice:**
- For any task expected to require more than 15 turns, decompose it into sessions before starting. Define the output of each session — a written design document, a set of implemented files, a debugging hypothesis to test — so that each session has a clear deliverable that can be passed to the next session as concrete context.[^6]
- Use written artifacts as session handoffs: at the end of a session, ask the model to generate a structured summary of decisions made, constraints established, and the specific starting point for the next session. This summary is better than the compact output for cross-session handoff because it is formatted as starting context for a fresh session rather than as a summary of a completed conversation.[^3]
- In the session log, group related sessions by task: "Architecture analysis (session 1 of 3): explored current implementation, identified coupling points, documented [output]. Session 2 will: ..." This grouping maintains the task-level view across multiple sessions and provides the documentation chain for understanding a complex task's evolution.[^4]
- Evaluate multi-session task architecture as a standard approach for complex work rather than a fallback for sessions that run too long. A planned three-session workflow for a complex feature is more predictable in cost, quality, and timeline than a single-session attempt that becomes a four-session workflow when the first session degrades and requires a restart.

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Early Signal Detection | Add periodic context fidelity checks to sessions longer than 10 turns | Engineering team |
| Proactive Compact Strategy | Plan compact points before multi-phase sessions; review compact summaries | Engineering team |
| Reset vs. Compact Decision | Document reset decision criteria in team session norms | Architect |
| Multi-Session Architecture | Identify complex tasks that benefit from planned multi-session design | Architect |

---

[^1]: Anthropic — "Managing Long Sessions," Claude Code Documentation, 2026. https://code.claude.com/docs/en/managing-long-sessions
    Context degradation mechanism; `/compact` mechanics and timing; session reset vs. compact decision criteria; context window attention behavior as the session lengthens.

[^2]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    Early degradation signals; precision decline as a degradation proxy; multi-session workflow architecture for complex tasks; written artifact handoffs between sessions.

[^3]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    Context fidelity testing patterns; pre-compact note-taking habit; extracting useful output from failed sessions before reset; structured session handoff document format.

[^4]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    Proactive compact timing at phase boundaries; compact summary review as a quality gate; turn-count soft limits by session type; pre-planned compact points for multi-phase sessions.

[^5]: The Pragmatic Engineer — "AI Tooling for Software Engineers in 2026," March 2026. https://newsletter.pragmaticengineer.com/p/ai-tooling-2026
    Session reset as the intervention for accumulated incorrect state; sunk cost reasoning as the primary cause of continuing bad sessions; the compact vs. reset decision framework applied to common session failure types.

[^6]: Anthropic — "2026 Agentic Coding Trends Report," Anthropic, 2026. https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf
    Multi-session workflow architecture as a standard approach for complex tasks; session scope and termination criteria for long tasks; cost and quality comparison between planned multi-session and degraded single-session approaches.
