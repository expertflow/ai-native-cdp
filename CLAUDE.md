# CLAUDE.md — AI-Native CDP (Claude Code)

Claude-specific shim for this project. The AI-agnostic working agreement lives in [AGENTS.md](AGENTS.md) — read it first; this file only adds Claude-specific configuration on top.

## Session initialization

1. Read [AGENTS.md](AGENTS.md) — canonical rules for all AI tools.
2. Read [.ai/memory/MEMORY.md](.ai/memory/MEMORY.md) — project memory index; apply any remembered context before responding.

## Project memory

Project memories go in `.ai/memory/` (AI-agnostic, readable by any tool).
- Write new memories as `.ai/memory/<type>-<slug>.md` and add a one-line pointer to `.ai/memory/MEMORY.md`.
- Never write project-specific memories to the global `~/.claude/` store.

## Claude-specific tool rules

- **Jira & Confluence** — use the `mcp__claude_ai_Atlassian__*` tools. Projects: **CRM** and **CIM**.
- **GitLab** (`https://gitlab.expertflow.com`) — use the `gitlab` MCP server (`mcp__gitlab__*`). Never use `glab` CLI or raw `curl`/`fetch`.
- **Google Chat** — post updates to space `spaces/AAAAyffa-Sk` using the `gws-chat-send` skill.
