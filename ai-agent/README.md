# ai-agent — Universal Engineering Orchestrator Suite

Platform-agnostic multi-agent setup for a Node.js/TypeScript repository. The
core structure and the MCP config in `mcp.json` work unmodified across Cursor,
VS Code Copilot, and Claude Code/Desktop.

## Layout

- `mcp.json` — canonical MCP server definitions (universal source of truth).
- `skills/` — SKILL.md files in Direction / Blueprints / Solutions (DBS) format.
- `agents/` — agent definition files used by the orchestration flow.

## Wiring per platform (copy the same `mcpServers` block)

- **Claude Code**: create `.mcp.json` at the repo root with the contents of
  `mcp.json`; skills are loaded from `.claude/skills/` (symlink/copy the suite).
- **Cursor**: copy the `mcpServers` block into `.cursor/mcp.json`; skills live
  under `.cursor/rules/`.
- **VS Code Copilot**: copy the `mcpServers` block into `.vscode/mcp.json`;
  skills live under `.github/` prompt files or `.copilot/` context.

Only `mcp.json` and the workspace-root path in the `filesystem` server args are
platform-specific. The skills and agents themselves are not.

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
