## Runbook Standards for AI-Assisted Operations

**Related to:** [Documentation Overview](00-overview.md) — Area 2: Runbook Standards · [Governance: Incident Response](../Governance/07-incident-response.md)[^a] · [Issues: Security Vulnerabilities](../Issues/04-security-vulnerabilities.md)[^b] · [Tooling: Hooks and Automation](../Tooling & Configuration/02-hooks-and-automation.md)[^c] · [QA & Testing: QA Engineer Workflow](../QA%20&%20Testing/05-qa-engineer-workflow.md)[^d]

---

## Overview

Runbooks are operational procedures — step-by-step instructions for deploying the system, executing data migrations, diagnosing and recovering from specific failure modes, and performing routine operational tasks. In pre-AI development, runbooks were written for human engineers: they could be somewhat ambiguous, assume contextual knowledge, and rely on the engineer's judgment to navigate edge cases. An engineer reading a runbook would fill in the gaps from experience and adapt the steps to the actual system state.[^1]

Claude Code changes what runbooks are for. A runbook that is precise enough for a human expert to follow is also a script that Claude Code can read and execute — step by step, confirming preconditions, reporting results, and producing a full execution log. This is not a marginal improvement in operational efficiency; it is a qualitative shift in how the team can respond to incidents. A 3 AM production incident handled by an engineer following a Claude-executed runbook that reads the relevant playbook, checks preconditions, and performs each step — while producing a log that the team can review the next morning — is fundamentally different from the same incident handled by an exhausted engineer following a runbook from memory.[^2]

But this capability requires that runbooks be written to a higher standard of precision than human-oriented runbooks typically achieve. Ambiguous steps that human engineers interpret with judgment are ambiguous steps that Claude Code may execute incorrectly. The good news is that the precision standard that makes runbooks Claude-executable is the same standard that makes them safe for a junior engineer to follow under pressure — the act of making runbooks machine-parseable makes them better human runbooks as well.

---

## Section 1: Runbook Format That Claude Code Can Execute

**Description:** A Claude Code-executable runbook has specific structural requirements that distinguish it from a general-purpose procedure document. Each step must be atomic: it performs exactly one observable action and produces exactly one verifiable result. Preconditions must be explicit: the conditions that must be true before the step begins, stated in terms that can be checked programmatically or via a specific command. Expected outputs must be specified: what the engineer (or Claude Code session) should observe after a successful step, so that deviation from the expected output triggers a pause rather than proceeding to the next step on potentially incorrect state. Failure paths must be documented: for each step that can fail, what to do — not "contact your team lead" but the specific recovery action or the specific escalation command to run.[^3]

Steps that are too vague for safe Claude Code execution — "ensure the database is healthy," "check that the deployment succeeded" — are also too vague for a new engineer following the runbook during an incident. The runbook writing discipline that produces Claude-executable procedures is identical to the discipline that produces reliable human-following procedures. The test is not "would an expert understand this step?" but "would a non-expert in a high-stress situation perform this step safely and know when it had succeeded?"

**Recommended Practice:**
- Adopt a consistent runbook step format: **Step N: [action verb + object].** Precondition: [what must be true]. Command: [exact command or UI action]. Expected output: [what success looks like, including specific strings or states to observe]. On failure: [specific recovery step or escalation target].[^3]
- Include a runbook header with metadata that Claude Code can parse: **Runbook:** [title], **System:** [affected system], **Last verified:** [date], **Verified by:** [engineer name], **Prerequisite access:** [list of permissions required before starting]. Claude Code checks this header before beginning execution to confirm it has the required access and that the runbook is current.[^5]
- Write command steps with exact commands, including flags and environment variables, rather than parametric descriptions. "Run the migration script" is not a runbook step; "`NODE_ENV=production node scripts/migrate.js --env production --dry-run`" is. The dry-run flag and exact flags are part of the operational safety — not implementation details that the executor should determine.[^1]
- Test runbooks in a staging environment before adding them to the runbook library, and include the test result as part of the runbook header: **Staging verified:** [date] by [engineer]. Untested runbooks are the source of most runbook-following failures — the engineer discovers during a production incident that step 4 assumes a tool version that was not installed.

---

## Section 2: Using Claude Code in Runbook Execution

**Description:** Claude Code runbook execution is not just writing runbooks with AI — it is using Claude Code as the operational agent that reads a runbook and performs its steps. This requires a specific session pattern: the engineer opens a Claude Code session, loads the relevant runbook (from the repository or via Google Drive MCP), issues the execution command, and monitors the session as it works through each step. The engineer is not passive — they confirm high-risk steps before Claude executes them, monitor for unexpected states, and can halt the session at any point. But Claude is doing the execution work, not the engineer.[^2]

The session discipline for runbook execution is different from the session discipline for code generation. In execution sessions, the primary risk is not producing incorrect code that fails at review — it is performing irreversible actions on production systems. The session must be configured conservatively: only the permissions required for the specific runbook steps, explicit confirmation prompts before any destructive or irreversible step, and a log of every action taken that can be reviewed after the session completes.[^7]

**Recommended Practice:**
- Define a standard runbook execution command that the team uses consistently: "Read the runbook at [path/MCP reference]. Before executing any step, confirm the preconditions are met. For any step marked DESTRUCTIVE or IRREVERSIBLE, pause and request explicit confirmation before proceeding. After each step, confirm the expected output is observed before proceeding to the next step. If the expected output is not observed, halt and report."[^2]
- Configure Claude Code sessions for runbook execution with explicit permission scoping: only the permissions required for the runbook's specific steps. A database migration runbook execution session does not need write access to the deployment configuration. Over-permissioned execution sessions are the primary risk surface for runbook execution errors.[^7]
- Produce a runbook execution log as a standard session output: the log records each step executed, the command run, the output observed, whether the expected output was matched, and any deviation from the expected path. Store the execution log alongside the runbook in Google Drive for the 30 days following execution — it is the audit trail for operational changes and the primary debugging resource if a post-execution issue is discovered.[^5]
- Limit Claude Code runbook execution to runbooks that have been marked as execution-verified: runbooks that have been successfully executed by Claude Code in a staging environment at least once, with an engineer monitoring. Unverified runbooks may have steps that are ambiguous in ways that only surface during machine execution — discover these in staging, not production.

---

## Section 3: Keeping Runbooks Current as AI-Generated Code Changes Procedures

**Description:** Runbooks become outdated when the systems they describe change. In pre-AI development, systems changed slowly enough that a runbook review on a biannual schedule was approximately sufficient. In AI-assisted development, systems can change significantly in a single sprint — and the engineer who made the change may not have thought about the runbook implications, particularly if the change was AI-generated and the engineer is operating at AI-output speed rather than human-deliberation speed.[^8]

The runbook maintenance failure mode is discovered at the worst possible time: during a production incident, when an engineer follows a runbook that describes a system that no longer exists. The deployment procedure in the runbook references a CI/CD pipeline step that was refactored two sprints ago; the database migration procedure references a schema that was changed in an AI-generated PR last week; the authentication recovery procedure references an endpoint that was renamed. Each of these is a runbook gap that accumulated silently during routine development and becomes visible only under operational pressure.

**Recommended Practice:**
- Require runbook review as part of the merge criteria for any AI-primary PR that modifies a component covered by an existing runbook. The PR author is responsible for identifying which runbooks (if any) are affected and either updating them or attesting that no runbooks require update. The PR template includes a checkbox: "I have reviewed the runbook library for affected runbooks and either updated them or confirmed no updates are required."[^8]
- Maintain a runbook-to-component index: a simple table that maps each runbook to the system components it covers. When an engineer reviews a PR that modifies one of those components, the index surfaces the relevant runbooks without requiring the engineer to remember that a runbook exists. Store this index in `docs/runbooks/README.md` alongside the ADR index.[^3]
- Schedule a monthly runbook verification pass as part of the engineering team's operational hygiene: the QA engineer and one backend engineer each pick two runbooks and verify that the procedures match the current system by walking through the steps in a staging environment. This practice ensures that runbooks are exercised regularly rather than only during incidents.
- When an AI-assisted code change makes a runbook incorrect and the correction is discovered during an incident, treat the runbook gap as a process failure: conduct a brief post-incident review specifically about why the runbook was not updated when the relevant code changed, and add a preventive check to the PR template for similar component types.[^9]

---

## Section 4: Runbook Location and MCP Accessibility

**Description:** A runbook that exists but cannot be found during an incident is operationally equivalent to one that does not exist. Runbook discoverability is an operational requirement that most teams treat as a documentation preference — the result is runbooks scattered across wikis, repositories, and individual engineers' note-taking systems, with no canonical location that an engineer under pressure knows to consult. For AI-assisted teams, the discoverability problem has an additional dimension: Claude Code needs to be able to find the relevant runbook at session time, which requires a queryable location accessible from a session.[^5]

The Google Drive runbook library with MCP accessibility is the recommended approach for this team: it is accessible from Claude Code sessions, it is accessible by engineers from any device during an incident, and it is the same documentation platform as the ADR library, reducing the number of knowledge stores the team maintains. The naming convention and folder structure must support both human search (an engineer typing "deployment" into Drive search during an incident) and MCP query (a Claude Code session asking "what is the runbook for production deployment?").[^10]

**Recommended Practice:**
- Establish a dedicated Google Drive folder for runbooks: `Engineering Runbooks/[System]/[Runbook Name]`. The folder structure mirrors the system components it covers — `Engineering Runbooks/Deployment/Production Deployment`, `Engineering Runbooks/Database/Migrations`, `Engineering Runbooks/Authentication/Recovery`. An engineer or Claude Code session can navigate to the relevant runbook by system and operation type.[^5]
- Configure the Google Drive MCP server with the Engineering Runbooks folder in the accessible paths. Add a CLAUDE.md instruction: "For any operational task involving deployment, migration, infrastructure change, or incident response, check the Engineering Runbooks folder in Google Drive before beginning. Use the relevant runbook as the procedure for the session."[^10]
- Maintain a runbook quick-reference card in the repository root as `RUNBOOKS.md`: a single table listing runbook names, the Drive link, and a one-line description. This card is accessible from the repository when engineers do not have Drive access or when a session needs to identify the right runbook quickly.[^3]
- Review the runbook library location and accessibility at the quarterly engineering health review: can runbooks be found within 30 seconds by any team member? Can Claude Code sessions find and load them via MCP? Is the naming convention consistent across all runbooks? Address any discoverability failures before they accumulate.[^8]

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| Runbook Format | Adopt structured step format with preconditions, commands, expected outputs, failure paths | Backend lead |
| Execution Pattern | Define standard runbook execution command; configure conservative permission scoping | Architect |
| Execution Logging | Produce and store execution logs for 30 days following runbook execution | Backend lead |
| Currency Maintenance | Add runbook review checkbox to AI-heavy PR template; build runbook-to-component index | Backend lead + Architect |
| Monthly Verification | Schedule monthly runbook verification pass with QA engineer and backend engineer | QA engineer |
| MCP Accessibility | Configure Google Drive MCP for runbook folder; add CLAUDE.md instruction; create RUNBOOKS.md | Architect |

---

[^1]: Ravikanth Konda — "Human-AI Collaboration in Software Teams: Evaluating Productivity, Quality, and Knowledge Transfer with Agentic and LLM-Based Tools," *International Journal of AI, BigData, Computational and Management Studies*, February 17, 2026. https://ijaibdcms.org/index.php/ijaibdcms/article/view/418
 Runbook precision requirements in AI-assisted operations: the standard that makes procedures safe for machine execution also makes them better human-following procedures under pressure.

[^2]: Boris Cherny at Y Combinator — "Inside Claude Code With Its Creator Boris Cherny," February 17, 2026. https://www.ycombinator.com/library/NJ-inside-claude-code-with-its-creator-boris-cherny
 Claude Code as an operational execution agent: the agentic session pattern for runbook execution; the engineer's monitoring role during execution sessions; why session execution is qualitatively different from session-assisted writing.

[^3]: Addy Osmani — "My LLM Coding Workflow Going Into 2026," April 2026. https://addyosmani.com/blog/ai-coding-workflow/
 Precision requirements for AI-parseable procedure documentation; the runbook-to-component index as a discoverability mechanism; the test of a good runbook step as "non-expert under pressure."

[^5]: Anthropic — "Model Context Protocol," Anthropic, 2025. https://www.anthropic.com/news/model-context-protocol
 MCP accessibility for operational documentation: configuring Google Drive MCP for runbook discoverability; the runbook header metadata format that allows sessions to verify currency before execution.

[^7]: Anthropic — "Security and Permissions," Claude Code Documentation, 2026. https://code.claude.com/docs/en/security-permissions
 Permission scoping for operational execution sessions: how to configure Claude Code sessions with only the permissions required for specific runbook steps; the `--dangerouslySkipPermissions` flag and why it should never appear in runbook execution sessions.

[^8]: Kyros — "The Vibe Coding Crisis: How AI-Generated Technical Debt Is Costing Companies Millions," March 2026. https://usekyros.ai/blog/vibe-coding-crisis-ai-technical-debt
 Operational documentation debt as a category of AI-assisted development debt: how AI-generated code changes accumulate runbook gaps that surface as incident failures; runbook review as a PR merge criterion.

[^9]: Roman Fedytskyi — "A Safer CI Pattern for Agentic Code Review," Medium, March 2026. https://medium.com/@roman_fedyskyi/a-safer-ci-pattern-for-agentic-code-review-94a484b5e3c4
 Post-incident runbook gap analysis: treating outdated runbooks discovered during incidents as process failures rather than documentation failures; preventive PR template additions.

[^10]: Dex Horthy (YC Root Access) — "Advanced Context Engineering for Agents," YouTube, August 2025. https://www.youtube.com/watch?v=IS_y40zY-hc
 Runbook MCP accessibility and CLAUDE.md integration: how to configure sessions to discover and load runbooks via Google Drive MCP; the CLAUDE.md instruction that makes runbook lookup automatic for operational sessions.

[^a]: [Governance: Incident Response](../Governance/07-incident-response.md) — runbooks are the operational artifact incident response procedures depend on; the two documents are written and maintained together.
[^b]: [Issues: Security Vulnerabilities](../Issues/04-security-vulnerabilities.md) — security incident runbooks are the primary operational countermeasure to the vulnerability accumulation documented here.
[^c]: [Tooling: Hooks and Automation](../Tooling & Configuration/02-hooks-and-automation.md) — hooks can trigger runbook validation steps; automated checks in the pipeline enforce runbook currency requirements.
[^d]: [QA & Testing: QA Engineer Workflow](../QA%20&%20Testing/05-qa-engineer-workflow.md) — QA engineers are primary runbook consumers; their workflow defines what runbook detail level is operationally sufficient.
