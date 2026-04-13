# ClaudeCodeReview

A collection of documentation and templates describing how to use Claude Code effectively on a small engineering team — including functional limitations, governance models, workflow patterns, tooling configuration, and ethics considerations.

**Team context:** All documents are written for a team of 11: 4 backend engineers, 3 frontend engineers, 1 architect, 1 QA engineer, and 2 product managers.

**DISCLAIMER:** All documents are AI generated. All sources were found using ClaudeCode to research. The claims in them are cited and the maintainers of this repository will have checked the claims alongside the citations provided by Monday April 13th. Do not think of the documents as anything but scaffolding around these citations.

---

## Quick Start

If you are new to this repository, start here:

1. **[Quality and Safety/Issues/](Quality%20and%20Safety/Issues/)** — Understand the failure modes before adopting AI-assisted development at scale
2. **[Core Practice/Workflows/](Core%20Practice/Workflows/)** — Learn the session patterns that produce reliable output
3. **[Core Practice/Tooling & Configuration/](Core%20Practice/Tooling%20%26%20Configuration/)** — Configure all Claude Code configuration surfaces: CLAUDE.md, settings, hooks, MCP, skills, agents, rules, and memory
4. **[Reference Artifacts/](Reference%20Artifacts/)** — Copy the actual configuration artifacts into your project

---

## Folder Index

### [Core Practice/](Core%20Practice/)

| Folder | What it covers |
|---|---|
| [Workflows/](Core%20Practice/Workflows/) | Session patterns: Explore-Plan-Implement, task decomposition, agentic delegation, parallel agent coordination |
| [Prompting/](Core%20Practice/Prompting/) | Prompt architecture, anti-patterns, context injection, iterative refinement, prompt library |
| [Tooling & Configuration/](Core%20Practice/Tooling%20%26%20Configuration/) | CLAUDE.md hierarchy, settings & permissions, hooks, MCP, commands & skills, agents, rules, memory, keybindings, CI/CD |
| [MCP Servers/](Core%20Practice/MCP%20Servers/) | GitHub, Slack, Google Drive, Linear, database, filesystem, and custom server integrations |

### [Quality and Safety/](Quality%20and%20Safety/)

| Folder | What it covers |
|---|---|
| [Issues/](Quality%20and%20Safety/Issues/) | Comprehension debt, codebase bloat, architectural drift, security vulnerabilities, review theater, skill atrophy |
| [Security/](Quality%20and%20Safety/Security/) | Threat modeling, SAST/DAST integration, dependency security, secrets management, vulnerability response |
| [QA & Testing/](Quality%20and%20Safety/QA%20%26%20Testing/) | Test session design, AI-generated test coverage, acceptance criteria automation, regression prevention |
| [Metrics/](Quality%20and%20Safety/Metrics/) | AI code quality, health dashboard, rework rate, security trends, session efficiency, codebase health |

### [Team Operations/](Team%20Operations/)

| Folder | What it covers |
|---|---|
| [Governance/](Team%20Operations/Governance/) | Review policies, AI usage policy, sprint gates, escalation procedures, quarterly health, compliance, incident response |
| [Learning/](Team%20Operations/Learning/) | Engineer onboarding, skill maintenance, team knowledge sharing, AI-free practice, continuing education |
| [Ethics/](Team%20Operations/Ethics/) | Intellectual property, developer impact, security responsibility, training data privacy, environmental costs, bias |
| [Documentation/](Team%20Operations/Documentation/) | Architecture decision records, runbook standards, knowledge transfer, rationale capture |

### [Economics and Reliability/](Economics%20and%20Reliability/)

| Folder | What it covers |
|---|---|
| [Cost & Token Economics/](Economics%20and%20Reliability/Cost%20%26%20Token%20Economics/) | Token cost mechanics, model selection strategy, context optimization, budget governance, cost monitoring |
| [Debugging & Troubleshooting/](Economics%20and%20Reliability/Debugging%20%26%20Troubleshooting/) | Session failure recognition, hallucination verification, context degradation, session recovery, escalation criteria |

### [Reference Artifacts/](Reference%20Artifacts/)

| Folder | What it covers |
|---|---|
| [Reference Artifacts/](Reference%20Artifacts/) | Ready-to-use CLAUDE.md, .mcp.json, settings.json, spec.md, PR template, and slash command examples |

---

## File Conventions

Each folder contains:
- `00-overview.md` — Full topic overview with all sub-topic summaries and a recommended actions table
- `01-`, `02-`, ... — Individual sub-topic files with detailed sections, recommended practices, and citations

All cited sources are from 2025–2026. YouTube citations include approximate timestamps.
