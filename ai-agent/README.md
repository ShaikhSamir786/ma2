# ai-agent — Universal Engineering Orchestrator Suite

Platform-agnostic multi-agent setup for a Node.js/TypeScript repository. The
core structure and the MCP config in `mcp.json` work unmodified across Cursor,
VS Code Copilot, and Claude Code/Desktop.

## Layout

- `mcp.json` — canonical MCP server definitions (universal source of truth).
- `skills/` — SKILL.md files in Direction / Blueprints / Solutions (DBS) format.
- `agents/` — agent definition files used by the orchestration flow.

## Wiring per platform (links, zero copies)

`ai-agent/` owns every byte; platform folders are bare links to it.

| Tool | Rules/context | Skills | Agents | MCP |
|---|---|---|---|---|
| Claude Code | `CLAUDE.md` (root, hardlink → `AGENTS.md` → `rules.md`) | `.claude/skills/` (junction → `ai-agent/skills`) | `.claude/agents/` (junction → `ai-agent/agents`) | `.mcp.json` (root, hardlink → `mcp.json`) |
| opencode | `AGENTS.md` (root) → `rules.md` | `.claude/skills/` (same junction — opencode scans it) | — | `opencode.jsonc` (root, `mcp` block, opencode schema) |
| GitHub Copilot (VS Code) | `AGENTS.md` + `CLAUDE.md` (both auto-detected) | `.claude/skills/` (same junction — Copilot scans it) | `.claude/agents/` (same junction — Copilot reads Claude-format agents) | `.vscode/mcp.json` (hardlink → `mcp.json`) |
| Antigravity (IDE + CLI + AGY) | `AGENTS.md` + `GEMINI.md` (both parsed) | `.agents/skills/` (junction → `ai-agent/skills`) | `.agents/agents/` (junction → `ai-agent/agents`) | `.agents/mcp_config.json` (hardlink → `mcp.json`) |
| Cursor | `.cursor/rules/` (add `.mdc` rules if used) | — | — | `.cursor/mcp.json` (hardlink → `mcp.json`) |
| VS Code Copilot | `.github/` prompt files or `.copilot/` context | — | — | `.vscode/mcp.json` (copy the `mcpServers` block) |

Only the `filesystem` server's workspace-root path in `mcp.json` and the per-tool
config shims are platform-specific. The skills and agents themselves are not.

## Tool mapping for the orchestrator flow

| Agent / skill need | Native tool | MCP server fallback |
|---|---|---|
| Read / Grep / Glob (repo-analyzer) | engine-native | `filesystem` (read_file, search_files, directory_tree) |
| Bash (verifier, repo-analyzer) | engine-native | `shell-commands` (execute_command) |
| Docs lookup for unfamiliar frameworks | — | `fetch` |
| Issue/PR/code search (optional) | — | `github` (needs GITHUB_TOKEN) |

## The flow

```
repo-analyzer (discovery)
        ↓
task → relevant domain skill(s)  →  senior-engineer-mentor (real decisions)
        ↓
implementation (smallest safe change)
        ↓
verifier (repo's own format/lint/test/build)
        ↓
code-reviewer (substantive changes)
        ↓
plain report
```

Scale the flow to the task; see `skills/engineering-orchestrator/SKILL.md`.
