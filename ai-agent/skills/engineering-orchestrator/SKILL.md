---
name: engineering-orchestrator
description: >-
  Governs any substantive engineering task in this repository — feature work,
  bug fixes, performance diagnosis, deployment, architecture review. Triggers on
  "add feature X", "fix this bug", "why is this slow", "deploy this",
  "review the architecture", or any request that implies acting on the actual
  codebase (expected stack: Node.js/TypeScript). Does NOT trigger for general
  questions unrelated to this code. Routes through repo-analyzer (discovery),
  domain skills (software-architect, backend-engineer, frontend-engineer,
  database-engineer, devops-engineer, security-engineer, qa-engineer,
  observability-engineer, rag-engineer, ai-engineer, ai-agent-engineer,
  universal-package-updater), senior-engineer-mentor (pre-implementation
  judgment), code-reviewer (post-implementation review), and verifier
  (post-implementation checks).
---

# Engineering Orchestrator

This is the coordination layer. It contains no implementation knowledge, no
judgment logic, and no review logic — those live in the domain skills,
senior-engineer-mentor, and code-reviewer. Its only job: figure out what is
actually in this repository, then route the task to the smallest correct set of
capabilities in the right order.

## Direction

Goal state: the task is completed with the smallest safe change that satisfies
it, verified by this repo's own commands, reviewed for any substantive change,
and reported plainly.

Constraints the orchestrator must respect:

- Never assume a stack, tool, or command. Expected Node.js/TypeScript, but the
  profile comes from repo-analyzer evidence, never from guesswork.
- Use only the skills and agents that change the outcome of this specific task.
- Never duplicate what a delegated skill already owns (mapping, not re-implementation).
- Do not write files outside the task scope. Do not run unsafe commands.
- Verify with the repo's own commands before declaring any change complete.

## Blueprints

Deterministic routing flow, scaled to task size:

1. **Discover** — delegate to repo-analyzer first unless this conversation
   already holds a current repo profile. Re-run discovery when the task enters
   an unseen part of the repo or observed reality contradicts the profile.
2. **Map the task to domain skills by detected technology** —
   backend framework (NestJS/Express/Fastify) → backend-engineer;
   frontend framework (React/Next.js/Vite) → frontend-engineer;
   schema/query/migration work → database-engineer;
   CI/CD/Docker/infra → devops-engineer;
   auth/input-handling/attack surface → security-engineer;
   test strategy/coverage → qa-engineer;
   observability/alerting → observability-engineer;
   multi-component structure → software-architect;
   LLM/agent/RAG code → ai-engineer / ai-agent-engineer / rag-engineer;
   dependency updates/audits → universal-package-updater.
   Never activate a skill "because it exists" — select the minimum set the task
   and detected stack actually require.
3. **Judgment gate** — for any real design/technology/trade-off decision (not
   mechanical work like a typo fix or version bump), route through
   senior-engineer-mentor before deciding.
4. **Implement** — smallest safe change that correctly and completely satisfies
   the task, preserving existing conventions, contracts, and deployment
   assumptions. No unrelated refactoring, no incidental dependency or infra
   changes.
5. **Verify** — delegate to verifier (format/lint/test/build using the repo's
   own commands only). Treat the result as ground truth; on failure, fix and
   re-verify rather than reporting success anyway.
6. **Review** — for a meaningful implementation, run code-reviewer after
   verification passes, with the repo profile as context.
7. **Report** — plain-language summary of what was found and changed; no
   narration of routing mechanics.

**Scale the flow:**

- **Trivial** (typo, one-line config, version bump): discover if the stack is
  unknown, make the change, run a cheap relevant check if one exists, done. No
  mentor, no full review.
- **Moderate** (bug fix, small single-area feature): discover, one or two
  domain skills, implement, verify, light review pass.
- **Substantial** (cross-layer feature, architecture change, production
  readiness): full flow, including senior-engineer-mentor for real decisions
  and code-reviewer at the end.

## Solutions

Acceptance criteria for success:

- The task is complete and demonstrably satisfies the request.
- Every command run came from the repo's own definitions (package.json scripts,
  lockfile-detected package manager, Makefile) — never a generic guess.
- verifier returned passed, or partial with every failure fixed and re-verified.
- No unrelated files were touched; no conventions were silently changed.
- Output is a plain-language result, not orchestration mechanics.

## Runtime Constraints and Boundary Checks

**STOP AND ASK when:**

- repo-analyzer returns low-confidence or `not detected` on something the task
  needs — investigate further or ask, never fabricate an answer.
- A command's safety or correctness for this repo is uncertain — check before
  running, not after.
- A technology is genuinely unfamiliar and the repo offers no pattern to follow
  — say so plainly, prefer the repo's own conventions over general assumptions.
- Verification fails and the cause is not identifiable — look for a genuine
  cause before retrying blindly.
- A lockfile mismatch appears (e.g. npm vs pnpm) — flag it; do not silently
  pick one and migrate.

**NEVER:**

- Fabricate a stack, command, or result.
- Run destructive commands (migrations against real data, deploys, force-push).
- Claim success when verification did not pass.
- Spin up skills or agents that cannot change the outcome.
- Change the package manager or introduce a new framework pattern without a
  task-tied reason.

## Extensibility

Adding support for a new technology means updating the mapping in Blueprints
step 2 (or, more often, nothing at all — most new frameworks fall under an
existing domain skill) and possibly deepening repo-analyzer's detection signals.
Never rewrite this skill, the domain skills, or the agents for a single
framework.

## Core Principle

One system, not one giant agent and not a combinatorial library of per-framework
agents. The domain skills already generalize across their area; this skill's job
is purely to figure out what is real in this repository and route to the
smallest set of existing capabilities that correctly handles it.
