# Workspace Rules

Global context for AI agents working in this workspace. Load this file first.

## What this is

A workspace hosting a Universal Engineering Orchestrator Suite — a
platform-agnostic multi-agent setup that works across Cursor, VS Code Copilot,
and Claude Code/Desktop. Expected project stack: Node.js/TypeScript.

## Where things live

- `ai-agent/mcp.json` — universal MCP server config (source of truth; wire the
  same `mcpServers` block into `.cursor/mcp.json`, `.vscode/mcp.json`, or
  `.mcp.json` per platform).
- `ai-agent/skills/` — 15 skills in Direction / Blueprints / Solutions (DBS)
  format, loaded on trigger.
- `ai-agent/agents/` — `repo-analyzer.md` (discovery) and `verifier.md`
  (verification).
- `ai-agent/README.md` — full suite documentation and wiring guide.

## Mandated workflow

For any substantive engineering task, do not skip straight to implementation.
Route through the suite:

1. `engineering-orchestrator` — the coordination skill; selects the correct
   subset of the flow for the task size.
2. `repo-analyzer` — evidence-based stack discovery. Never assume a stack,
   package manager, test command, or framework.
3. Relevant domain skill(s) only — `software-architect`, `backend-engineer`,
   `frontend-engineer`, `database-engineer`, `devops-engineer`,
   `security-engineer`, `qa-engineer`, `observability-engineer`, `rag-engineer`,
   `ai-engineer`, `ai-agent-engineer`, `universal-package-updater`. Activate
   only what the task actually needs.
4. `senior-engineer-mentor` — before any real design/technology/trade-off
   decision, not for mechanical work.
5. Implement the smallest safe change, preserving existing conventions.
6. `verifier` — run the repo's own format/lint/test/build commands. Treat the
   result as ground truth; never report success when verification failed.
7. `code-reviewer` — for any meaningful implementation, after verification
   passes.

## Stack rule

Expected stack is Node.js/TypeScript, but always confirm from evidence:
determine the package manager from the lockfile (`package-lock.json` → npm,
`pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `bun.lockb` → bun) and use the
repo's actual scripts — never a generic guess.

## Verification rule

Never declare a task done without running the repo's own commands. If
`repo-analyzer` or `verifier` returns low-confidence or ambiguous results, say
so and investigate or ask — do not fabricate an answer.

## Safety

- Never run destructive commands: migrations against real data, deploys,
  force-push, `rm` on tracked work.
- Never commit or log secrets, tokens, or credentials.
- Never change the package manager or introduce new framework patterns without
  a task-tied reason.

## MCP

Servers are defined in `ai-agent/mcp.json` (`filesystem`, `shell-commands`,
`fetch`, `github`). The `github` server is optional and requires a
`GITHUB_PERSONAL_ACCESS_TOKEN` (env `GITHUB_TOKEN`); the other three run with
no credentials.
