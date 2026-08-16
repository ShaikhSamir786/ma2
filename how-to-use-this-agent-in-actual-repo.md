# How to Use This Agent Suite in an Actual Product Repo

This is the hands-on guide for taking the Universal Engineering Orchestrator
Suite from this workspace (`ma2`) and wiring it into a **real product
repository** so that opencode, Claude Code, GitHub Copilot (VS Code),
Antigravity, and Cursor all get the same 15 skills, 4 agents, and MCP servers —
with **zero duplication** (one canonical copy, platform folders are links).

It assumes:
- **Host OS:** Windows, PowerShell 5.1 (NTFS). Junctions for directories,
  hardlinks for files — no admin rights needed.
- **Product repo:** a fresh Node.js/TypeScript repo, not created yet.
- **Distribution choice:** copy the suite into the product repo (re-sync when
  the suite evolves).
- **Tools:** all of opencode, Claude Code, GitHub Copilot, Antigravity, Cursor.

`<REPO>` below means the absolute path of your product repo, e.g.
`S:\my-product`. Set it once in PowerShell:

```powershell
$repo = "S:\my-product"
```

---

## How the suite works (30-second mental model)

`ai-agent/` is the **single physical home** of everything:

```
<REPO>\
├── AGENTS.md               pointer → ai-agent/rules.md     (opencode, Cursor, Copilot)
├── CLAUDE.md               hardlink → AGENTS.md             (Claude Code)
├── GEMINI.md               hardlink → AGENTS.md             (Antigravity)
├── opencode.jsonc          opencode MCP config (opencode schema)
├── .mcp.json               hardlink → ai-agent/mcp.json     (Claude Code MCP)
├── ai-agent/               CANONICAL — the only real content
│   ├── rules.md            full global context (loaded first by every tool)
│   ├── mcp.json            MCP servers (filesystem, shell-commands, fetch, github)
│   ├── skills/             15 × <name>/SKILL.md (DBS format)
│   └── agents/             repo-analyzer.md, verifier.md,
│                           secret-scanner.md, dependency-auditor.md
├── .claude/
│   ├── skills/             junction → ai-agent/skills
│   └── agents/             junction → ai-agent/agents
├── .agents/
│   ├── skills/             junction → ai-agent/skills
│   ├── agents/             junction → ai-agent/agents
│   └── mcp_config.json     hardlink → ai-agent/mcp.json     (Antigravity MCP)
├── .cursor/mcp.json        hardlink → ai-agent/mcp.json     (Cursor MCP)
├── .vscode/mcp.json        hardlink → ai-agent/mcp.json     (VS Code Copilot MCP)
└── scripts/setup-ai-agent.ps1   recreates every link after a fresh clone
```

The skill files and agents are **not product-specific** — they discover the
actual stack from evidence (`repo-analyzer`) and run the repo's own commands
(`verifier`). Nothing in `ai-agent/` changes between the suite repo and your
product repo except the `filesystem` MCP root path.

---

## Phase 0 — Bootstrap the product repo

Create the repo and give the agents something real to detect:

```powershell
New-Item -ItemType Directory -Path $repo -Force
Set-Location $repo
git init
npm init -y
```

Add a `scripts` block to `package.json` so `verifier` has real commands to run:

```jsonc
"scripts": {
  "test": "node --test",
  "lint": "eslint .",
  "build": "tsc -p tsconfig.json"
}
```

Create a `.gitignore` (see Phase 5 for the full recommended content).

---

## Phase 1 — Copy the canonical suite into the product repo

Copy **only `ai-agent/`** — do not copy `.claude/`, `.agents/`, `.cursor/`,
`.vscode/`, `.mcp.json`, `CLAUDE.md`, or `GEMINI.md`. Those are links that
point at `ai-agent/` paths inside this workspace and will not transfer; you
recreate them in Phases 2–4.

```powershell
Copy-Item -Recurse -Force "S:\testing-viral\ma2\ai-agent" "$repo\ai-agent"
```

The copy contains everything the tools need:

| Content | Location |
|---|---|
| Global context | `ai-agent/rules.md` |
| MCP config (canonical) | `ai-agent/mcp.json` |
| 15 skills (DBS format) | `ai-agent/skills/<name>/SKILL.md` |
| 4 agents | `ai-agent/agents/` (`repo-analyzer`, `verifier`, `secret-scanner`, `dependency-auditor`) |

**One edit after copying** — set the `filesystem` MCP root to the product repo:

```powershell
$mcp = "$repo\ai-agent\mcp.json"
(Get-Content -Raw $mcp).Replace("S:/testing-viral/ma2", ($repo -replace '\\','/')) |
  Set-Content -Path $mcp -Encoding UTF8
```

---

## Phase 2 — Global context pointers (rules/context)

The one source of truth is `ai-agent/rules.md`. Every tool loads a root file
that points to it.

1. **`AGENTS.md`** — a thin pointer (write it fresh, this exact content):

   ```markdown
   # AGENTS.md

   Pointer file. The canonical global context for this workspace lives in
   `ai-agent/rules.md` — read that file first and follow it.
   ```

   Auto-loaded by: opencode, GitHub Copilot (VS Code), Cursor, Antigravity.

2. **`CLAUDE.md`** — hardlink to `AGENTS.md` (Claude Code auto-loads it):

   ```powershell
   New-Item -ItemType HardLink -Path "$repo\CLAUDE.md" -Target "$repo\AGENTS.md"
   ```

3. **`GEMINI.md`** — hardlink to `AGENTS.md` (Antigravity parses it):

   ```powershell
   New-Item -ItemType HardLink -Path "$repo\GEMINI.md" -Target "$repo\AGENTS.md"
   ```

Hardlinks are used instead of copies so the three files can never drift.
(Symlinks need admin rights on Windows even with Developer Mode on — hardlinks
and junctions do not.)

---

## Phase 3 — MCP servers for all 5 tools

MCP must be registered **per tool before the session starts** — a model cannot
"read" `mcp.json` into existence. Four tools share the Claude-style
`mcpServers` schema and can hardlink the same canonical file. opencode uses a
different schema and gets its own config shim.

### 3a. Claude Code — `.mcp.json`

```powershell
New-Item -ItemType HardLink -Path "$repo\.mcp.json" -Target "$repo\ai-agent\mcp.json"
```

### 3b. Cursor — `.cursor/mcp.json`

```powershell
New-Item -ItemType Directory -Path "$repo\.cursor" -Force | Out-Null
New-Item -ItemType HardLink -Path "$repo\.cursor\mcp.json" -Target "$repo\ai-agent\mcp.json"
```

### 3c. VS Code GitHub Copilot — `.vscode/mcp.json`

```powershell
New-Item -ItemType Directory -Path "$repo\.vscode" -Force | Out-Null
New-Item -ItemType HardLink -Path "$repo\.vscode\mcp.json" -Target "$repo\ai-agent\mcp.json"
```

### 3d. Antigravity — `.agents/mcp_config.json`

```powershell
New-Item -ItemType Directory -Path "$repo\.agents" -Force | Out-Null
New-Item -ItemType HardLink -Path "$repo\.agents\mcp_config.json" -Target "$repo\ai-agent\mcp.json"
```

### 3e. opencode — `opencode.jsonc` (opencode schema)

Different schema (`type: "local"`, `command` as array, `environment` instead of
`env`), so a small config shim at the root:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "filesystem": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-filesystem", "S:/my-product"]
    },
    "shell-commands": {
      "type": "local",
      "command": ["npx", "-y", "@g0t4/mcp-server-commands"]
    },
    "fetch": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-fetch"]
    },
    "github": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-github"],
      "environment": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "{env:GITHUB_TOKEN}"
      }
    }
  }
}
```

Replace `S:/my-product` with your product repo path. The `github` server stays
inert until `GITHUB_TOKEN` is set in your environment.

### What each server backs

| Server | Provides | Backs |
|---|---|---|
| `filesystem` | read/write/search workspace | Read / Grep / Glob in repo-analyzer |
| `shell-commands` | `execute_command` | Bash / verifier |
| `fetch` | web docs lookup | unfamiliar framework docs |
| `github` (optional) | repo/issue/PR/code search | needs `GITHUB_TOKEN` |

---

## Phase 4 — Skills + agents (junctions, not copies)

Skill and agent loaders scan fixed paths only; they cannot be redirected. The
`.claude/` path is the cross-tool standard (Claude Code, Copilot, opencode scan
it); `.agents/` serves Antigravity (and opencode).

```powershell
# Claude Code + GitHub Copilot + opencode
New-Item -ItemType Directory -Path "$repo\.claude" -Force | Out-Null
New-Item -ItemType Junction -Path "$repo\.claude\skills" -Target "$repo\ai-agent\skills"
New-Item -ItemType Junction -Path "$repo\.claude\agents" -Target "$repo\ai-agent\agents"

# Antigravity + opencode
New-Item -ItemType Junction -Path "$repo\.agents\skills" -Target "$repo\ai-agent\skills"
New-Item -ItemType Junction -Path "$repo\.agents\agents" -Target "$repo\ai-agent\agents"
```

**Cursor caveat:** Cursor does **not** scan skills or Claude-format agents. It
benefits from the suite via the MCP servers and `AGENTS.md` global context only.
Optionally add `.cursor/rules/*.mdc` rules if you want Cursor-specific behavior.

**Junction targets must be absolute.** A relative target like
`../ai-agent/skills` is wrong and breaks.

---

## Phase 5 — Git strategy (canonical-only + a setup script)

Junctions and hardlinks are OS/NTFS constructs. If you commit them, git stores
them as duplicate file copies, and on any other machine they are broken.
Recommended: **commit only canonical content**, ignore the link files, and
recreate the links with one script.

### `.gitignore` additions

```gitignore
# ai-agent platform adapters (recreated by scripts/setup-ai-agent.ps1)
.claude/
.agents/
.cursor/
.vscode/
.mcp.json
CLAUDE.md
GEMINI.md
opencode.jsonc
```

> Note: `opencode.jsonc` is machine-repo-specific (it embeds the repo path), so
> it is generated by the script, not committed. `AGENTS.md` and `ai-agent/`
> ARE committed — they are real content, not links.

### `scripts/setup-ai-agent.ps1` — recreate everything after a clone

Save this in the repo so a fresh clone self-wires with one command. It is
idempotent (re-run safely).

```powershell
# scripts/setup-ai-agent.ps1 — recreate all ai-agent platform links (Windows)
# Usage: powershell -ExecutionPolicy Bypass -File scripts/setup-ai-agent.ps1
$ErrorActionPreference = "Stop"
$root = Split-Path -Parent $PSScriptRoot
$agent = Join-Path $root "ai-agent"

function Link-File([string]$path, [string]$target) {
  if (Test-Path -LiteralPath $path) {
    Remove-Item -LiteralPath $path -Force
  }
  New-Item -ItemType HardLink -Path $path -Target $target | Out-Null
  Write-Host "  hardlink  $path -> $target"
}

function Link-Dir([string]$path, [string]$target) {
  if (Test-Path -LiteralPath $path) {
    Remove-Item -LiteralPath $path -Force
  }
  New-Item -ItemType Directory -Path (Split-Path -Parent $path) -Force | Out-Null
  New-Item -ItemType Junction -Path $path -Target $target | Out-Null
  Write-Host "  junction  $path -> $target"
}

Write-Host "Wiring ai-agent suite for all tools in: $root"

# Global context pointers
Link-File (Join-Path $root "CLAUDE.md") (Join-Path $root "AGENTS.md")
Link-File (Join-Path $root "GEMINI.md") (Join-Path $root "AGENTS.md")

# MCP (hardlinks — same file everywhere)
Link-File (Join-Path $root ".mcp.json")                    (Join-Path $agent "mcp.json")
Link-File (Join-Path $root ".cursor\mcp.json")             (Join-Path $agent "mcp.json")
Link-File (Join-Path $root ".vscode\mcp.json")             (Join-Path $agent "mcp.json")
Link-File (Join-Path $root ".agents\mcp_config.json")      (Join-Path $agent "mcp.json")

# Skills + agents (junctions)
Link-Dir (Join-Path $root ".claude\skills") (Join-Path $agent "skills")
Link-Dir (Join-Path $root ".claude\agents") (Join-Path $agent "agents")
Link-Dir (Join-Path $root ".agents\skills") (Join-Path $agent "skills")
Link-Dir (Join-Path $root ".agents\agents") (Join-Path $agent "agents")

# opencode shim (regenerate with the current repo path)
$mcpCanonical = Get-Content -Raw (Join-Path $agent "mcp.json")
$filesystemArgs = ""
$slash = $root -replace '\\', '/'
$opencode = @(
  "{"
  '  "$schema": "https://opencode.ai/config.json",'
  '  "mcp": {'
  '    "filesystem": { "type": "local", "command": ["npx", "-y", "@modelcontextprotocol/server-filesystem", "' + $slash + '"] },'
  '    "shell-commands": { "type": "local", "command": ["npx", "-y", "@g0t4/mcp-server-commands"] },'
  '    "fetch": { "type": "local", "command": ["npx", "-y", "@modelcontextprotocol/server-fetch"] },'
  '    "github": { "type": "local", "command": ["npx", "-y", "@modelcontextprotocol/server-github"], "environment": { "GITHUB_PERSONAL_ACCESS_TOKEN": "{env:GITHUB_TOKEN}" } }'
  "  }"
  "}"
) -join "`n"
Set-Content -Path (Join-Path $root "opencode.jsonc") -Value $opencode -Encoding UTF8
Write-Host "  wrote     opencode.jsonc (filesystem root: $slash)"

Write-Host "`nDone. Verify with: powershell -File scripts/setup-ai-agent.ps1 -Verbose"
```

Run it:

```powershell
Set-Location $repo
powershell -ExecutionPolicy Bypass -File scripts/setup-ai-agent.ps1
```

---

## Phase 6 — Verification

Run these from the product repo root and confirm every check passes:

```powershell
$root = $PWD

# 1. mcp.json is valid JSON (canonical + every hardlink reads the same)
Get-Content -Raw "$root\ai-agent\mcp.json" | ConvertFrom-Json | Out-Null
Get-Content -Raw "$root\.mcp.json" | ConvertFrom-Json | Out-Null
Get-Content -Raw "$root\.cursor\mcp.json" | ConvertFrom-Json | Out-Null

# 2. All 15 skills reachable through every adapter
(Get-ChildItem "$root\.claude\skills" -Directory).Name      # = ai-agent/skills listing
(Get-ChildItem "$root\.agents\skills" -Directory).Name      # same

# 3. Agents reachable
Get-ChildItem "$root\.claude\agents" -Name                  # repo-analyzer, verifier, ...
Get-ChildItem "$root\.agents\agents" -Name                  # same

# 4. Links are links, not copies
Get-Item "$root\.claude\skills", "$root\.claude\agents", "$root\.agents\skills", `
         "$root\.agents\agents", "$root\.mcp.json", "$root\.cursor\mcp.json", `
         "$root\CLAUDE.md", "$root\GEMINI.md" |
  Select-Object Name, LinkType, @{n='Target';e={$_.Target}}

# 5. Global context chain intact
Get-Content "$root\AGENTS.md"          # pointer
Get-Content "$root\ai-agent\rules.md"  # canonical

# 6. MCP registers per tool (start a session, then)
#    claude mcp list       -> 4 servers (filesystem, shell-commands, fetch, github)
#    opencode mcp list     -> same 4
```

First time Claude Code opens the repo it shows a workspace-trust prompt to
approve the project-scoped `.mcp.json` servers — expected, not an error.

---

## Using the suite day-to-day

The mandated workflow is encoded in `ai-agent/rules.md` and driven by
`engineering-orchestrator`. For any substantive task the agents will route:

```
task
  → repo-analyzer        (evidence-based discovery — stack, pkg manager, tests)
  → relevant domain skill (software-architect, backend/frontend/database/
                           devops/security/qa/observability/rag/ai/...)
  → senior-engineer-mentor (pre-implementation judgment on real trade-offs)
  → implementation        (smallest safe change, existing conventions)
  → verifier              (repo's OWN format/lint/test/build — ground truth)
  → code-reviewer         (post-implementation review)
  → plain report
```

Example prompts in the product repo:

- "Add user authentication to the API" → orchestrator routes through
  `backend-engineer` + `security-engineer`, then verifier + code-reviewer.
- "Why is this query slow?" → `database-engineer`.
- "Review this PR" / "how can I improve this code" → `code-reviewer`.
- "Update our dependencies" → `universal-package-updater`.
- "Deploy this service" → `devops-engineer`.

The flow scales to the task — trivial changes skip most of it; the
orchestrator decides.

---

## Keeping the suite updated

When `ma2`'s suite evolves, re-sync just the canonical content:

```powershell
Copy-Item -Recurse -Force "S:\testing-viral\ma2\ai-agent" "$repo\ai-agent"
```

Then re-apply the `filesystem` root path edit (Phase 1) if you copied over it,
and re-run `scripts/setup-ai-agent.ps1` (idempotent). If you find yourself
re-syncing many repos, switch to a git submodule pinned to the suite repo
instead of copies.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| "Administrator privilege required" on a link command | You used a symlink. Use a **hardlink** (files) or **junction** (dirs) instead — no admin needed. |
| A skill isn't listed in the tool | The loader scans fixed paths only. Confirm the junction exists (Phase 4) and the name is a directory under `ai-agent/skills/`. |
| MCP server not showing | MCP is registered per tool before the session. Check `.mcp.json` / `.cursor/mcp.json` / `.vscode/mcp.json` / `.agents/mcp_config.json` / `opencode.jsonc` are present and parse as JSON. Restart the session. |
| `github` server inert | Set `GITHUB_TOKEN` in your environment; it is optional. |
| Skills registered twice | Both `.claude/skills` and `.agents/skills` are junctions to the same bytes; tools scan them independently — harmless. |
| Junction/skill broken after clone | Links are not in git by design. Run `powershell -ExecutionPolicy Bypass -File scripts/setup-ai-agent.ps1`. |
