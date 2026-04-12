# Templates — Ready-to-Use Artifacts

This folder contains **working files you copy directly into your project**, not documentation about them.
Each file is production-ready with bracket placeholders where customization is required.

---

## What's Here vs. What's in Documentation/

| Folder | What It Contains |
|---|---|
| `Templates/` | Actual config files, prompt files, and templates you drop into repos |
| `Documentation/` | Explanations, rationale, and team policies |

If you're setting up a new project or onboarding Claude Code, start here.

---

## How to Use These Templates

1. **Copy** the file to its destination path (see table below).
2. **Search for `[` brackets** — every placeholder is wrapped in `[UPPER_SNAKE_CASE]`.
3. **Replace** each placeholder with your project's actual value.
4. **Commit** the file into your repo so the whole team shares the same context.

Some files (`.mcp.json`, `settings.json`) belong in your home directory or project root and are
git-ignored by default. The table below notes which ones to keep out of version control.

---

## Artifact Reference

| File | Destination in Your Project | Purpose | Commit to Repo? |
|---|---|---|---|
| `CLAUDE.md` | `./CLAUDE.md` (project root) | Persistent context Claude reads at session start: stack, conventions, off-limits patterns, corrections | Yes |
| `.mcp.json` | `~/.mcp.json` or `./.mcp.json` | MCP server wiring — GitHub, Slack, Linear, Drive | No (contains env var refs; add to .gitignore) |
| `settings.json` | `./.claude/settings.json` | Hooks (lint, SAST, context injection), permissions, tool allow-list | Yes (review carefully first) |
| `spec.md` | `./specs/[TICKET]-[slug].md` | Session input file — defines what Claude should build in a single session | Yes |
| `pull_request_template.md` | `./.github/pull_request_template.md` | GitHub PR template with AI-involvement and comprehension fields | Yes |
| `commands/security-review.md` | `./.claude/commands/security-review.md` | Slash command: `/security-review` — OWASP audit of current changes | Yes |
| `commands/feature-scaffold.md` | `./.claude/commands/feature-scaffold.md` | Slash command: `/feature-scaffold` — scaffold files from a spec | Yes |
| `commands/pr-description.md` | `./.claude/commands/pr-description.md` | Slash command: `/pr-description` — generate PR description from diff | Yes |

---

## Quick Setup for a New Project

```bash
# From your project root
cp path/to/Templates/CLAUDE.md ./CLAUDE.md
mkdir -p .claude/commands
cp path/to/Templates/settings.json ./.claude/settings.json
cp path/to/Templates/commands/*.md ./.claude/commands/
cp path/to/Templates/pull_request_template.md ./.github/pull_request_template.md

# Home-directory artifacts (shared across all your projects)
cp path/to/Templates/.mcp.json ~/.mcp.json
```

Then open each file, search for `[`, and fill in every placeholder before running Claude Code.

---

## Placeholder Convention

All placeholders use the format `[UPPER_SNAKE_CASE]`. Common ones across files:

| Placeholder | What to Put There |
|---|---|
| `[YOUR_PROJECT_NAME]` | e.g., `acme-api` |
| `[YOUR_STACK]` | e.g., `Node.js 20 / React 18 / PostgreSQL 15` |
| `[TICKET_NUMBER]` | e.g., `PROJ-412` |
| `[GITHUB_ORG]` | Your GitHub organization slug |
| `[LINEAR_TEAM_ID]` | Your Linear workspace team identifier |
| `[SLACK_CHANNEL_ID]` | Slack channel ID (not name) for notifications |

---

## Keeping Templates in Sync

When the team agrees on a new convention (e.g., a new off-limits pattern), update `Templates/CLAUDE.md`
**and** the `CLAUDE.md` in any active project repos. The Documentation folder holds the rationale;
the Templates folder holds the enforced rule.
