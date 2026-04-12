# ClaudeCodeReview

A collection of documentation and templates describing how to use Claude Code effectively on a small engineering team — including functional limitations, governance models, workflow patterns, tooling configuration, and ethics considerations.

**Team context:** All documents are written for a team of 11: 4 backend engineers, 3 frontend engineers, 1 architect, 1 QA engineer, and 2 product managers.

**DISCLAIMER:** All documents are AI generated. All sources were found using ClaudeCode to research. The claims in them are cited and the maintainers of this repository will have checked the claims alongside the citations provided by Monday April 13th. Do not think of the documents as anything but scaffolding around these citations.

---

## Quick Start

If you are new to this repository, start here:

1. **[Issues/](Issues/)** — Understand the failure modes before adopting AI-assisted development at scale
2. **[Workflows/](Workflows/)** — Learn the session patterns that produce reliable output
3. **[Tooling & Configuration/](Tooling%20%26%20Configuration/)** — Configure all Claude Code configuration surfaces: CLAUDE.md, settings, hooks, MCP, skills, agents, rules, and memory
4. **[Templates/](Templates/)** — Copy the actual configuration artifacts into your project

---

## Folder Index

### Core Practice

| Folder | What it covers |
|---|---|
| [Workflows/](Workflows/) | Session patterns: Explore-Plan-Implement, task decomposition, agentic delegation, parallel agent coordination |
| [Prompting/](Prompting/) | Prompt architecture, anti-patterns, context injection, iterative refinement, prompt library |
| [Tooling & Configuration/](Tooling%20%26%20Configuration/) | CLAUDE.md hierarchy, settings & permissions, hooks, MCP, commands & skills, agents, rules, memory, keybindings, CI/CD |
| [MCP Servers/](MCP%20Servers/) | GitHub, Slack, Google Drive, Linear, database, filesystem, and custom server integrations |

### Quality and Safety

| Folder | What it covers |
|---|---|
| [Issues/](Issues/) | Comprehension debt, codebase bloat, architectural drift, security vulnerabilities, review theater, skill atrophy |
| [Security/](Security/) | Threat modeling, SAST/DAST integration, dependency security, secrets management, vulnerability response |
| [QA & Testing/](QA%20&%20Testing/) | Test session design, AI-generated test coverage, acceptance criteria automation, regression prevention |
| [Metrics/](Metrics/) | AI code quality, health dashboard, rework rate, security trends, session efficiency, codebase health |

### Team Operations

| Folder | What it covers |
|---|---|
| [Governance/](Governance/) | Review policies, AI usage policy, sprint gates, escalation procedures, quarterly health, compliance, incident response |
| [Learning/](Learning/) | Engineer onboarding, skill maintenance, team knowledge sharing, AI-free practice, continuing education |
| [Ethics/](Ethics/) | Intellectual property, developer impact, security responsibility, training data privacy, environmental costs, bias |
| [Documentation/](Documentation/) | Architecture decision records, runbook standards, knowledge transfer, rationale capture |

### Economics and Reliability

| Folder | What it covers |
|---|---|
| [Cost & Token Economics/](Cost%20&%20Token%20Economics/) | Token cost mechanics, model selection strategy, context optimization, budget governance, cost monitoring |
| [Debugging & Troubleshooting/](Debugging%20&%20Troubleshooting/) | Session failure recognition, hallucination verification, context degradation, session recovery, escalation criteria |

### Reference Artifacts

| Folder | What it covers |
|---|---|
| [Templates/](Templates/) | Ready-to-use CLAUDE.md, .mcp.json, settings.json, spec.md, PR template, and slash command examples |

---

## File Conventions

Each folder contains:
- `00-overview.md` — Full topic overview with all sub-topic summaries and a recommended actions table
- `01-`, `02-`, ... — Individual sub-topic files with detailed sections, recommended practices, and citations

All cited sources are from 2025–2026. YouTube citations include approximate timestamps.
