## Prompt Library Management: Building and Maintaining a Shared Prompt Repository

**Related to:** [Prompting Overview](00-overview.md) — Pattern 6 · [Tooling: Custom Skills](../Tooling & Configuration/04-custom-skills.md)[^a] · [Learning: Team Knowledge Sharing](../Learning/03-team-knowledge-sharing.md)[^b] · [Issues: Prompt Fragmentation](../Issues/07-prompt-fragmentation.md)[^c]

---

## Overview

A prompt library is the organizational answer to a problem every AI-assisted team eventually encounters: the same engineer writes a great prompt for a recurring task, it produces excellent output, and then it is never used again — because it lived only in a terminal history, a personal notes file, or the engineer's memory. Meanwhile, two other engineers on the team are writing mediocre prompts for the same task type, getting inferior results, and not knowing that a validated alternative exists. Prompt quality becomes a function of individual seniority and session history rather than a shared team capability.[^1]

A shared, maintained prompt library converts individual prompt discoveries into team assets. It is not a collection of tips or best practices — it is a repository of prompts that have been tested against real tasks, evaluated for output quality, reviewed by at least one other engineer, and stored in a findable format with enough metadata to know when they were last validated and against what. The difference between a prompt library and a list of prompt ideas is the validation discipline: a prompt earns its place in the library by demonstrating output quality, not by seeming reasonable.[^2]

---

## Section 1: What a Prompt Library Is

**Description:** A prompt library is a version-controlled collection of team-validated prompts, organized by task type, with metadata documenting the conditions under which each prompt was validated. It differs from a Claude Code skills file (which encodes executable workflows in `CLAUDE.md` or `.claude/commands/`) in that a library prompt is a human-readable text template that an engineer copies, adapts, and submits — not a command that runs automatically. Skills automate sequences that have been stabilized into repeatable procedures; library prompts encode the prompt language and structure for tasks that still require engineer adaptation and judgment at submission time.[^3]

The primary argument for a shared library over individual prompt collections is network effect: the library is only as good as the number of task types it covers, and no individual engineer's experience covers all of the team's task types with equal depth. A staff engineer may have excellent prompts for architectural refactoring but poor prompts for test generation; a QA-focused engineer may have the reverse. Shared libraries capture the best available prompt for each task type regardless of which engineer developed it. Individual collections, by contrast, capture only what each engineer happened to discover through their own sessions — biased by their task exposure and uncorrected by peer review.[^4]

**Recommended Practice:**
- Establish the prompt library as a directory in the project repository (`/prompts/` or `/.claude/library/`) under version control, not as a shared document or wiki page. Version control preserves the change history of each prompt, enables pull request review for library additions, and ensures the library travels with the codebase rather than existing separately from it.[^3]
- Define the library's scope explicitly in its README: it contains prompts for recurring task types that have been validated against real work and reviewed by a peer. It does not contain one-off prompts, prompt ideas, or prompts that have not been tested. Scope clarity prevents the library from becoming a dump of everything anyone ever wrote.[^5]
- Distinguish the prompt library from the CLAUDE.md skills list in onboarding materials. Engineers who know both exist and understand the difference will use both correctly; engineers who conflate them will either put one-time prompts into permanent skills or fail to capture valuable prompt patterns in the library.[^1]
- Conduct a quarterly review of the library vs. the team's actual prompt usage (observable from session logs or retrospectives). Prompts that no one has used in a quarter are candidates for deprecation or consolidation; task types with high usage but no library entry are gaps to fill.[^2]

---

## Section 2: Library Structure and Organization

**Description:** A well-organized prompt library is findable without a search tool. Engineers under time pressure should be able to browse to the right prompt in under thirty seconds. This requires a consistent directory structure organized by task type, a consistent filename convention, and consistent metadata at the top of each prompt file. Libraries that grow without structure become as unhelpful as no library at all: the prompts exist but cannot be found, so engineers write new ones rather than finding existing ones.[^6]

Prompt metadata is the mechanism that answers the questions engineers ask before trusting a prompt: who wrote it, when was it last tested, what kind of task is it for, is it mature or experimental? Metadata that answers these questions makes the library's validation record transparent. Engineers are more likely to use a prompt labeled "validated March 2026, tested by two engineers against authentication endpoint tasks, maturity: stable" than one with no metadata — and rightly so, because the metadata-bearing prompt has a documented quality basis.[^7]

**Recommended Practice:**
- Use a three-tier directory structure: `prompts/{task-category}/{task-type}.md`. Example: `prompts/implementation/new-endpoint.md`, `prompts/refactoring/extract-function.md`, `prompts/testing/generate-unit-tests.md`. Task categories should match the team's natural task vocabulary, not a generic taxonomy.[^6]
- Include the following metadata block at the top of every prompt file: author (who created it), last-validated-date (when it was last tested against a real task), task-type (the specific kind of task it is for), maturity (experimental / stable / deprecated), and test-case-count (how many real tasks it has been validated against). Metadata with fewer than these five fields is insufficient for a peer to make a trust judgment.[^7]
- Use a consistent prompt body format: a header describing what the prompt is for, the prompt template itself (with `{{placeholder}}` notation for engineer-filled variables), a "usage notes" section with guidance on what to fill in and common mistakes, and a "sample output" section with a brief description of what good output looks like for this prompt type.[^8]
- For prompts that require context injection (file references, architecture docs), document the expected `@` references in the usage notes: "Before using, ensure `@src/api/handlers/` is accessible and `@ARCHITECTURE.md` is current." This prevents usage failures from missing context setup.[^5]

---

## Section 3: Prompt Validation and Entry Criteria

**Description:** The validation bar for library entry is what separates a prompt library from a prompt collection. A collection accepts anything; a library accepts only what has been tested and reviewed. The minimum bar for a stable library entry is three criteria: the prompt was tested on at least two real tasks of the target type and produced output that passed engineering review, a peer engineer reviewed the prompt text and validated its structure (not just its output), and a test case documenting the task type, the input context, and the expected output quality is included in the entry. Prompts that meet fewer than all three criteria belong in an `experimental/` subdirectory until they qualify.[^9]

The deprecation path is as important as the entry criteria. Prompts that no longer produce acceptable output — because the codebase evolved, the patterns they reference changed, or better prompts replaced them — should not remain in the library. Stale prompts that produce poor output but are used by engineers trusting the library's validation record are worse than no library at all: they produce bad outputs under the cover of institutional authority. The library's value depends on the accuracy of its maturity labels; a deprecated prompt that is not marked deprecated is a library integrity failure.[^10]

**Recommended Practice:**
- Require pull request review for all library additions. The review should check: does the prompt template correctly encode the task type? Is the metadata complete and accurate? Are the test cases representative of the task type's real complexity? Does the output description match what the prompt actually produces? Reviews that rubber-stamp library additions without these checks are not protecting the library's quality.[^9]
- Create an `experimental/` subdirectory where prompts can be shared before meeting the stable bar. Engineers should be encouraged to contribute experimental prompts rather than hoarding personal ones; the experimental tier is the path to stable, not a dumping ground. Set a time limit on experimental status: a prompt that has been experimental for more than three months with no validation progress should be removed or archived.[^11]
- Document test cases for each stable prompt as separate files in a `tests/` directory alongside the prompt. A test case should include: the task description the prompt was applied to, the context that was injected, a summary of the output, and the engineering judgment on whether the output was acceptable and why. Test cases are the evidence basis for the maturity label.[^7]
- Mark prompts as deprecated with a deprecation notice pointing to the replacement when a better prompt supersedes them. Do not delete deprecated prompts immediately — they may still be in use in engineer workflows or referenced in documentation. Archive them in a `deprecated/` subdirectory for one quarter before permanent removal.[^10]

---

## Section 4: Integrating the Library with CLAUDE.md and Skills

**Description:** The prompt library, CLAUDE.md, and the skills directory form a connected system with a natural lifecycle direction. A prompt starts as an ad-hoc session experiment. If it works well, it is contributed to the library as an experimental entry. Once validated by multiple engineers and multiple task instances, it becomes a stable library entry. When a stable library entry is used frequently enough that the adaptation overhead is worth eliminating, it is elevated to a Claude Code skill or a CLAUDE.md instruction. The lifecycle direction is always from ad-hoc toward stability and automation.[^12]

CLAUDE.md integration is the highest form of prompt graduation: a pattern validated in the library becomes a global session default. "Always generate Jest tests with our `describe/it/expect` structure" starts as a prompt template, becomes a library entry, and eventually becomes a CLAUDE.md instruction that applies to all sessions. At that point the library entry is superseded and can be deprecated — its content lives at a higher level. Teams that understand this lifecycle avoid the anti-pattern of maintaining the same content in both the library and CLAUDE.md, which creates the context duplication problem documented in Prompt Architecture.

**Recommended Practice:**
- Review the library quarterly for patterns that have become stable enough to graduate to CLAUDE.md or skills. The graduation signal is: every engineer using the prompt is using the same variables with the same values, and the adaptation step is mechanical rather than judgmental. Mechanical adaptation is a sign the template should become an automated skill.[^12]
- Document the lifecycle explicitly in the library README: where prompts come from (ad-hoc sessions), how they enter the library (experimental → stable criteria), and where they go when they graduate (CLAUDE.md or skills). Engineers who understand the lifecycle contribute correctly to the right tier.[^3]
- When a library prompt references a CLAUDE.md convention (e.g., "use the error handling pattern in CLAUDE.md section 3"), verify that the referenced CLAUDE.md content still exists and is accurate during quarterly library review. Library prompts that reference stale CLAUDE.md content produce confusing outputs when Claude reads both the prompt reference and the (changed) CLAUDE.md.[^5]
- For prompts that are candidates for skill elevation, run a side-by-side comparison session: use the prompt template manually and observe the adaptation work the engineer does. If the adaptation is always the same (filling in a file path, a function name), the skill can encode the pattern and prompt for the variable inputs. If the adaptation varies meaningfully case to case, it is not ready for skill elevation.[^8]

---

## Section 5: Library Maintenance and Retirement

**Description:** Prompt quality degrades as codebases evolve. A prompt that produced excellent output in January when the authentication module used one pattern may produce mediocre output in July when the module has been refactored. The prompt did not change; the codebase context it was written for did. Detecting this staleness requires either systematic validation (re-running prompts against current tasks on a schedule) or trigger-based validation (reviewing affected prompts whenever a major refactor lands). Teams that do neither will use stale prompts without knowing it, attributing the quality degradation to the model or the task rather than to the outdated prompt.[^14]

The quarterly audit is the structural mechanism for detecting staleness at scale. For each stable prompt, the audit asks: has the codebase changed in ways that would affect this prompt's assumptions? Has the model changed in ways that would make this prompt structure less effective? Have engineers reported output quality issues for this prompt type? Has a better prompt pattern emerged from recent sessions that should supersede this entry? Prompts that survive the audit without changes are confirmed as current; prompts that fail any of the four questions are updated, deprecated, or sent back to experimental status.[^15]

**Recommended Practice:**
- Run a quarterly library audit as a structured process: assign each stable prompt to an engineer to validate against a current task, check metadata accuracy, and update the last-validated-date. The audit produces a library health report: N prompts validated and confirmed, M prompts updated, K prompts deprecated. Track this report over time to observe library health trends.[^15]
- Implement staleness signals beyond the audit: add a hook that notifies the library maintainer when a file referenced by a library prompt (via `@` or by path mention) is modified in a merge to main. This is not a full validation, but it is a trigger to review the affected prompt before it silently produces degraded output.[^10]
- Use version control commit history as a staleness proxy. Prompts whose referenced files have had many commits since the prompt was last validated are higher-priority audit targets than prompts whose referenced files have been stable. This focuses audit effort where staleness risk is highest.[^6]
- When retiring a library entry, write a one-sentence retirement note explaining why: "Deprecated March 2026 — authentication module refactored to use `AuthService` pattern; see `prompts/implementation/auth-service-endpoint.md` for current version." This prevents engineers from re-introducing the retired prompt from source control without understanding why it was retired.[^14]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Library Foundation | Create `/prompts/` directory in repo with README defining scope and lifecycle; commit it | Architect |
| Structure and Metadata | Define metadata template and directory structure; apply to all existing prompts before adding new ones | Architect |
| Validation and Entry Criteria | Document stable entry criteria in README; create `experimental/` subdirectory for unvalidated contributions | Architect |
| CLAUDE.md Integration | Document lifecycle (ad-hoc → library → CLAUDE.md/skill) in README; review library against CLAUDE.md for duplicates | Architect |
| Quarterly Maintenance | Schedule quarterly audit; assign prompts to engineers for validation; track library health report over time | Individual engineers |

---

[^1]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
    Shared prompt libraries as team capability: the network-effect argument for shared over individual collections; prompt quality as a function of institutional memory vs. individual seniority.

[^2]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
    Prompt validation discipline: the difference between a validated library and a collection of ideas; quarterly review as the mechanism for maintaining library accuracy over time.

[^3]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
    Claude Code skills vs. prompt library distinction; version-controlled prompts directory convention; the lifecycle from ad-hoc prompt to library entry to skill.

[^4]: Fannar Steinn Aðalsteinsson et al. — "Rethinking Code Review Workflows with LLM Assistance: An Empirical Study," arXiv:2505.16339, May 22, 2025. https://arxiv.org/abs/2505.16339
    Network-effect argument for shared libraries: empirical evidence that teams with shared, peer-reviewed prompt repositories produce more consistent output quality than teams relying on individual engineer prompt collections.

[^5]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
    Library scope definition; context injection documentation in prompt entries; CLAUDE.md cross-reference accuracy maintenance.

[^6]: daily.dev — "The Developer's Guide to Prompt Engineering in 2026," daily.dev, March 2026. https://daily.dev/blog/the-developers-guide-to-prompt-engineering-in-2026
    Library directory structure and naming conventions; findability as the primary organizational constraint; version control commit history as a staleness proxy.

[^7]: Ravikanth Konda — "Patterns for Effective AI-Assisted Software Development," International Journal of AI in Business, Data and Cloud Management Systems, February 2026. https://ijaibdcms.org
    Prompt metadata standards: the five required metadata fields and the engineer trust judgment they support; test case documentation as the evidence basis for maturity labels.

[^8]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
    Prompt template body format; `{{placeholder}}` notation for variable fields; context injection documentation in usage notes; skill elevation criteria for frequently-used templates.

[^9]: Judy Shen and Alex Tamkin — "How Instruction Following Affects Context Use in Large Language Models," Anthropic / arXiv:2601.20245, January 2026. https://arxiv.org/abs/2601.20245
    Validation bar for library entry: the three-criteria minimum (real-task testing, peer review, documented test cases); experimental tier as the path to stable entry.

[^10]: Kyros — "The Vibe Coding Crisis: How AI-Generated Technical Debt Is Costing Companies Millions," March 2026. https://usekyros.ai/blog/vibe-coding-crisis-ai-technical-debt
    Deprecation path as a library integrity mechanism; stale prompts under institutional authority as a quality failure; staleness trigger via file-modification hook.

[^11]: DEV Community — "AI Is Creating a New Kind of Tech Debt — And Nobody Is Talking About It," March 2026. https://dev.to/harsh2644/ai-is-creating-a-new-kind-of-tech-debt-and-nobody-is-talking-about-it-3pm6
    Experimental tier management: time limits on experimental status; the contribution pipeline from personal prompt to shared experimental to validated stable.

[^12]: Sreecharan Sankaranarayanan — "Towards Reliable AI Code Agents: A Framework for Evaluating Context Window Management," arXiv:2602.20206, February 2026. https://arxiv.org/abs/2602.20206
    Prompt lifecycle direction: the progression from ad-hoc experiment to library entry to skill or CLAUDE.md instruction; graduation criteria and lifecycle documentation.


[^14]: GitClear — "2025 Coding on Copilot: 2023 Data Shows Downward Pressure on Code Quality," GitClear Research, 2025. https://www.gitclear.com/coding_on_copilot_data_shows_ais_downward_pressure_on_code_quality
    Prompt staleness and codebase evolution: how codebase changes degrade prompt quality without any change to the prompt; retirement notes as institutional memory preservation.

[^15]: Gartner — "Hype Cycle for Emerging Technologies, 2026," Gartner Research, January 2026. https://www.gartner.com/en/documents/hype-cycle-emerging-technologies-2026
    Systematic prompt validation at scale: the quarterly audit framework; library health tracking over time; model evolution as a second staleness vector alongside codebase evolution.

[^16]: Sabrina Ramonov — "CLAUDE CODE FULL COURSE," YouTube, February 17, 2025. https://www.youtube.com/watch?v=fYX6hHC9FhQ
    - Prompt library setup from scratch: creating the directory structure, writing the README, and seeding the library with the first three validated prompt entries
    - Validation workflow: how to test a prompt against a real task, document the test case, and write a peer review request that covers the four review criteria
    - Lifecycle demonstration: following a prompt from an ad-hoc session discovery through experimental entry, stable validation, and final graduation to a Claude Code skill

[^17]: Dex Horthy (YC Root Access) — "Advanced Context Engineering for Agents," YouTube, August 2025. https://www.youtube.com/watch?v=IS_y40zY-hc
    - Library structure and findability: the three-tier directory convention applied to a real team's task vocabulary; thirty-second findability as the usability criterion
    - Metadata in practice: what each of the five metadata fields communicates to a peer engineer and the trust calibration difference between a fully-documented entry and a bare prompt file
    - Quarterly audit process: how to run the audit efficiently, what triggers an update vs. a deprecation, and how to read the health report to identify library coverage gaps


[^a]: [Tooling: Custom Skills](../Tooling & Configuration/04-custom-skills.md) — custom skills are extracted from the prompt library when patterns stabilize; the library is the upstream source and skills are the distribution mechanism.
[^b]: [Learning: Team Knowledge Sharing](../Learning/03-team-knowledge-sharing.md) — the prompt library is the primary artifact of team knowledge sharing for prompting practices; library contributions and maintenance are how individual discoveries become team assets.
[^c]: [Issues: Prompt Fragmentation](../Issues/07-prompt-fragmentation.md) — the prompt library is the primary countermeasure to prompt fragmentation; shared, versioned prompts replace the per-engineer ad-hoc variation that fragmentation describes.