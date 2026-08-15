# Build Playbook — Universal Engineering Orchestrator Suite

Complete, deep-dive record of how this repo was built, every decision made, the
exact commands that worked (and the ones that didn't), and a step-by-step
template to recreate a multi-agent suite like this from scratch.

---

## 1. What this repo is

`S:\testing-viral\ma2` hosts a **platform-agnostic multi-agent system**: 15
engineering skills + 2 agents + MCP configuration, all living in `ai-agent/`,
mapped into platform folders (`.claude/`, `.cursor/`) with **zero duplication**
via filesystem links.

| Component | Location | Count |
|---|---|---|
| Skills (DBS format) | `ai-agent/skills/<name>/SKILL.md` | 15 |
| Agents | `ai-agent/agents/` | 2 |
| MCP config (canonical) | `ai-agent/mcp.json` | 1 |
| Global context (canonical) | `ai-agent/rules.md` | 1 |
| Suite docs | `ai-agent/README.md`, this file | 2 |
| Platform adapters | `.claude/skills/` (junction), `.cursor/mcp.json` (hardlink), `CLAUDE.md` (hardlink) | 3 links, 0 copies |

---

## 2. The starting point (inputs we had)

Two raw inputs existed before anything was built:

1. **`mcp-multi-agent-architect.skill`** — a *skill archive* for the architect
   workflow that drives the whole build. It is a **ZIP archive**, not a plain
   file.
2. **`skills/` folder** — 17 more `.skill` files (also ZIPs) plus
   `agents/repo-analyzer.md` and `agents/verifier.md`.

### Critical fact #1: `.skill` files are ZIP archives

The Read tool rejects them as "binary". The signature bytes are `PK` (a ZIP).
To open one:

```powershell
# Expand-Archive refuses the .skill extension, so copy to .zip first
$tmp = "$env:TEMP\opencode\skill.zip"
Copy-Item ".\mcp-multi-agent-architect.skill" $tmp -Force
Expand-Archive -LiteralPath $tmp -DestinationPath "$env:TEMP\opencode\architect"
```

Inside you find `<name>/SKILL.md` (e.g. `mcp-multi-agent-architect/SKILL.md`).
Extract all of them in a loop:

```powershell
$root = "$env:TEMP\opencode\skills-src"
Get-ChildItem "skills" -Filter *.skill | ForEach-Object {
  Copy-Item $_.FullName $tmp -Force
  $d = Join-Path $root $_.BaseName
  Expand-Archive -LiteralPath $tmp -DestinationPath $d -Force
}
```

---

## 3. The architect skill's process (how the design was forced)

`mcp-multi-agent-architect/SKILL.md` mandates an **interview-then-generate**
workflow. This is the reusable pattern:

### Stage 1 — Discovery interview (4 questions, ask before generating)

1. **Target environment** — Claude-only (`.claude/`) vs Universal
   (`ai-agent/` with MCP hooks for Cursor/VS Code/Claude). → *We chose
   Universal.*
2. **Tech stack & language** — drives dependency rules inside every SKILL.md.
   → *Node.js/TypeScript.*
3. **Active intent & skills** — what workflows the agents must run. →
   *Engineering orchestrator suite.*
4. **Skill source material** — existing skill folder/prompt to ingest, or
   from scratch. → *The `skills/` folder.*

### Stage 2 — Synthesize and emit

Mandatory output shape:

1. **Visual workspace tree** (what will exist after the build).
2. **Core MCP config** (`ai-agent/mcp.json`) — fully realized JSON, server
   names + transport + the tools each exposes. No placeholders.
3. **Behavioral files** — every SKILL.md/agent in a deterministic
   **Direction / Blueprints / Solutions (DBS)** format plus runtime
   constraints, with absolute dependency and allowed-tools lists.

Rules that must hold for every generated file: no placeholders/TODOs; paths,
JSON, YAML internally consistent with the tree; nothing Claude-specific in the
core `ai-agent/` structure (vendor specifics only as optional blocks).

---

## 4. The DBS format (the skill file contract)

Every skill was rewritten from the source material into this exact shape:

```markdown
---
name: <kebab-case-name>
description: >-                      # algorithmic trigger keywords, NOT prose
  <what it owns / when it activates /
   what it does NOT own / stack rules>
---

# <Name>

## Direction                           # goal state + constraints
Goal state: ...
Constraints the skill must respect:
- ...
- Dependency rules (Node.js/TypeScript): ...

## Blueprints                          # deterministic ordered steps + decision gates
1. ...
2. ...
Decision gates:                        # "if X then do Y, else do Z"
- **Choice A vs B**: rule for deciding.

## Solutions                           # expected artifact + acceptance criteria
Expected output: ...
Acceptance criteria:
- ...

## Runtime Constraints and Boundary Checks
- **NEVER**: ...
- **STOP AND ASK when**: ...
- **STOP AND FLAG**: ...

## Interaction With Other Skills       # the orchestration graph
```

Why this works: **Direction** keeps the model honest about the goal and
boundaries; **Blueprints** give deterministic routing instead of freeform
reasoning; **Solutions** give measurable done-conditions; the constraints
blocklist prevents common failure modes.

### The 15 skills + 2 agents that resulted

| Skill | Owns |
|---|---|
| `engineering-orchestrator` | The coordination layer — routes to everything else, never implements |
| `software-architect` | Component boundaries, trade-offs, contracts |
| `backend-engineer` | APIs, business logic, auth, queues (NestJS/Express, BullMQ) |
| `frontend-engineer` | React/Next.js UI, state, rendering strategy |
| `database-engineer` | Schema, indexes, migrations, EXPLAIN-driven diagnosis |
| `devops-engineer` | Docker, CI/CD, deployment, infra right-sizing |
| `security-engineer` | Threat modeling, authz review, dependency vulns |
| `qa-engineer` | Test strategy, quality gates, regression tests |
| `observability-engineer` | Logs/metrics/traces, dashboards, alerting |
| `rag-engineer` | Retrieval pipelines, chunking, recall@k |
| `ai-engineer` | LLM integration, structured output, cost/latency |
| `ai-agent-engineer` | Agent loops, step limits, confirmation gates |
| `code-reviewer` | Post-implementation review (severity tiers) |
| `senior-engineer-mentor` | Pre-implementation judgment (mentor layer) |
| `universal-package-updater` | Dependency scans/updates/audits |
| `agents/repo-analyzer.md` | Evidence-based stack discovery (read-only) |
| `agents/verifier.md` | Runs the repo's own format/lint/test/build |

**Scope decision made:** `gbp-local-seo-strategist` and
`resume-jd-fit-scorer` were **excluded** as non-engineering (not part of the
orchestrator suite). Decide this per build — the suite only carries what the
intent needs.

---

## 5. MCP config design (`ai-agent/mcp.json`)

Four servers, all stdio, all `npx`-launched, none Claude-specific:

| Server | Package | Provides | Notes |
|---|---|---|---|
| `filesystem` | `@modelcontextprotocol/server-filesystem` | read/write/search workspace — backs Read/Grep/Glob | `args` includes the workspace path (the only machine-specific value) |
| `shell-commands` | `@g0t4/mcp-server-commands` | `execute_command` — backs Bash/verifier | |
| `fetch` | `@modelcontextprotocol/server-fetch` | web docs lookup | |
| `github` | `@modelcontextprotocol/server-github` | repo/issue/PR/code search | **optional**; needs `GITHUB_PERSONAL_ACCESS_TOKEN` |

```json
{
  "$schema": "https://modelcontextprotocol.io/schema/2025-06-18/schema.json",
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "S:/testing-viral/ma2"],
      "type": "stdio",
      "description": "...",
      "tools": ["read_file", "search_files", "directory_tree", "..."]
    }
  }
}
```

Pattern: each server gets a `tools` array listing what it exposes (this is a
declarative mapping required by the architect skill, even though the MCP spec
itself doesn't define it), and a `description` explaining which agent need it
backs.

---

## 6. Global context layering (`rules.md` / `AGENTS.md` / `CLAUDE.md`)

Three files, **one source of truth**:

1. **`ai-agent/rules.md`** — the CANONICAL full global context (suite layout,
   mandated workflow, stack rule, verification rule, safety, MCP).
2. **`AGENTS.md`** (repo root) — a **pointer**: "read `ai-agent/rules.md`
   first". opencode auto-loads this.
3. **`CLAUDE.md`** (repo root) — **hardlink to `AGENTS.md`** (same content,
   never drifts). Claude Code auto-loads this.

**Lesson:** do NOT keep two full copies of global context — they drift. One
canonical file + thin pointers. The pointer pattern is:

```markdown
# AGENTS.md

Pointer file. The canonical global context for this workspace lives in
`ai-agent/rules.md` — read that file first and follow it.
```

---

## 7. Platform wiring — the links (junctions vs symlinks vs hardlinks)

The `ai-agent/` folder is the single physical home of all content. Platform
folders only **link** to it — zero copies, zero drift.

### The rules on Windows (PowerShell 5.1, NTFS)

| Link type | PowerShell | Admin needed? | Good for |
|---|---|---|---|
| **Junction** (dir) | `New-Item -ItemType Junction` | No | whole folder (`.claude/skills` → `ai-agent/skills`) |
| **Hardlink** (file) | `New-Item -ItemType HardLink` | No | single file, same volume (`.cursor/mcp.json`, `CLAUDE.md`) |
| **Symlink** (file) | `New-Item -ItemType SymbolicLink` | **Yes, in practice** | file — *failed here even with Developer Mode on* |

### Commands that worked

```powershell
$root = "S:\testing-viral\ma2"

# directory junction: expose all 15 skills to Claude Code
New-Item -ItemType Directory -Path "$root\.claude" -Force | Out-Null
New-Item -ItemType Junction -Path "$root\.claude\skills" -Target "$root\ai-agent\skills"

# file hardlink: Cursor reads its mcp.json at this exact path
New-Item -ItemType Directory -Path "$root\.cursor" -Force | Out-Null
New-Item -ItemType HardLink -Path "$root\.cursor\mcp.json" -Target "$root\ai-agent\mcp.json"

# file hardlink: Claude Code loads CLAUDE.md
New-Item -ItemType HardLink -Path "$root\CLAUDE.md" -Target "$root\AGENTS.md"
```

### The failure we hit (learn from this)

`New-Item -ItemType SymbolicLink` for `.cursor\mcp.json` failed with:

```
Administrator privilege required for this operation.
NewItemSymbolicLinkElevationRequired
```

Even though Developer Mode was ON (`AllowDevelopmentWithoutDevLicense = 1`)
and `S:` is a fixed NTFS drive. The shell simply lacked
`SeCreateSymbolicLinkPrivilege`. **Fallback: use a hardlink for files** —
same volume, no privilege needed, identical zero-drift result.

### Pitfall: relative junction paths

Proposed `../../ai-agent/skills` is wrong. `.claude/` sits at the repo root, so
from `.claude/` the target is **one** level up: `../ai-agent/skills`. Absolute
targets avoid the whole class of error — the junction commands above used
absolute `$root\...` targets.

### Verify links resolved

```powershell
Get-Item "$root\.claude\skills" | Select-Object Name, LinkType, Target
(Get-Item "$root\.claude\skills").GetFiles().Count   # 0 = junction owns no bytes
Get-ChildItem "$root\.claude\skills" -Directory      # all 15 skills visible?
Get-Item "$root\.cursor\mcp.json" | Select-Object LinkType, Target
```

---

## 8. Cleanup — deleting the legacy inputs

After the suite was built and verified, the originals became redundant:

- `skills/` folder (17 archives + old agents) — **deleted** (its content was
  converted into `ai-agent/`).
- `mcp-multi-agent-architect.skill` — **deleted** (generator tool, only needed
  to rebuild).

```powershell
Remove-Item -Recurse -Force "S:\testing-viral\ma2\skills"
Remove-Item -Force "S:\testing-viral\ma2\mcp-multi-agent-architect.skill"
```

**Trade-off consciously made:** the two excluded skills
(`gbp-local-seo-strategist`, `resume-jd-fit-scorer`) were permanently lost
since we chose "Delete everything" with no backup. If you value irreversibility,
archive first (`_archive/` or zip) — the safer default is archive-then-delete.

---

## 9. Verification of the whole system

```powershell
# 1. mcp.json is valid JSON
Get-Content -Raw "ai-agent\mcp.json" | ConvertFrom-Json | Out-Null

# 2. All 15 skills reachable through the platform adapter
(Get-ChildItem ".claude\skills" -Directory).Name   # = ai-agent/skills listing

# 3. Links are links, not copies (link target check)
Get-Item ".claude\skills", ".cursor\mcp.json", "CLAUDE.md" | Select Name, LinkType, Target

# 4. Global context chain intact
Get-Content "AGENTS.md"      # pointer
Get-Content "ai-agent\rules.md"  # canonical
```

---

## 10. Recreate-from-scratch template (condensed)

Use this as your checklist next time:

1. **Have an architect skill** (a `.skill` ZIP containing a SKILL.md that
   mandates interview-first). Extract it to read its rules.
2. **Run the discovery interview** — environment, stack, intent, source
   material. Do not generate until answered.
3. **Ingest source material** — extract every `.skill` ZIP, read every
   `SKILL.md`, note what's in scope / out of scope.
4. **Design the tree** (plan mode first): `ai-agent/{mcp.json,rules.md,
   README.md,skills/,agents/}`.
5. **Write `mcp.json`** — stdio servers, npx-launched, `tools` arrays, one
   optional credentialed server flagged as such.
6. **Convert every skill to DBS** — Direction / Blueprints / Solutions /
   Constraints, with stack-specific dependency rules baked in.
7. **Write the canonical `rules.md`** — the full global context.
8. **Write root pointers** — `AGENTS.md` (pointer) + `CLAUDE.md` (hardlink).
9. **Wire platform adapters** — junction `.claude/skills`, hardlink
   `.cursor/mcp.json`. Use hardlinks for files (symlinks need admin), junctions
   for dirs, absolute targets.
10. **Verify** — JSON parses, links resolve, skills visible through adapters.
11. **Clean up legacy inputs** — archive first (safer), then delete.
12. **Document the process** — write a playbook like this one while context is
    fresh.

---

## 11. Key lessons (read these before next time)

1. **`.skill` files are ZIPs** — copy to `.zip`, then `Expand-Archive`.
2. **Interview before generate** — the architect skill's 4 questions prevent
   building the wrong thing.
3. **One canonical copy + links, never copies** — junctions/hardlinks for
   platform adapters; pointer files for global context. Zero drift.
4. **File symlinks need admin on Windows** even with Developer Mode — use
   **hardlinks** for files, **junctions** for directories.
5. **Relative link paths are error-prone** — use absolute targets.
6. **Scope deliberately** — exclude skills that don't match the intent
   (non-engineering ones here); say so explicitly so it's a decision, not an
   omission.
7. **Never assume a stack** — repo-analyzer-first is the core rule of the
   orchestrator; every generated skill embeds it.
8. **Archive before delete** — permanent deletion was fine here by explicit
   choice; archive-then-delete is the safer default.
9. **Document while fresh** — the working commands and the one failure
   (symlink) are exactly what a future re-build needs.

---

## Appendix A — Final file tree

```
S:\testing-viral\ma2\
├── AGENTS.md                 pointer → ai-agent/rules.md
├── CLAUDE.md                 hardlink → AGENTS.md
├── ai-agent/                 canonical source of truth
│   ├── rules.md              full global context
│   ├── mcp.json              MCP servers (filesystem, shell-commands, fetch, github)
│   ├── README.md             suite docs + per-platform wiring
│   ├── BUILD-PLAYBOOK.md     this file
│   ├── skills/               15 × <name>/SKILL.md (DBS format)
│   └── agents/               repo-analyzer.md, verifier.md
├── .claude/
│   └── skills/               junction → ai-agent/skills
└── .cursor/
    └── mcp.json              hardlink → ai-agent/mcp.json
```
