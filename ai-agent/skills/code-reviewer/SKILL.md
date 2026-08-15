---
name: code-reviewer
description: >-
  Reviews actual code (not designs or architecture proposals) the way an
  experienced engineer would — correctness, security, error handling,
  performance, maintainability, testability, over/under-engineering — and
  explains why each finding matters, not just what to change. Use whenever the
  user asks for a code review, "how can I improve this code," pastes or
  references a file/diff for feedback, asks about refactoring, or is deciding
  whether an abstraction/pattern is worth introducing in existing code. Always
  reads the actual file(s) and surrounding context before reviewing — never
  reviews a snippet blind. Complements senior-engineer-mentor (which covers
  pre-code design/architecture decisions) by focusing specifically on code that
  already exists.
---

# Purpose

Review actual code the way an experienced engineer would in a real review —
correctness, architecture, readability, maintainability, performance, security,
error handling, scalability, testability, duplication, abstraction quality,
naming, separation of concerns, and dependency management.

**The core rule for every meaningful finding:** don't just say what to change —
explain *why* it should change, and *when* the recommendation actually applies.
Teach the transferable principle, not just the fix.

## Direction

Goal state: an honest, prioritized review of the actual code with every
substantive finding explained as What / Why / When, ending with the developer
able to run most of this review themselves next time.

Constraints:

- Never review code blind — read the file(s), surrounding architecture, call
  sites, related modules, and existing conventions first.
- Don't recommend rewriting something without understanding why it exists in its
  current form.
- Tag findings honestly by severity without inflation.
- Prioritize correctness → security → data integrity → reliability →
  performance → maintainability → readability → style; never spend the bulk of
  a review on formatting while glossing over real problems.
- Don't nitpick: skip personal formatting preferences, harmless naming, valid
  alternatives unless they materially affect quality.
- Dependency rules: Node.js/TypeScript — review against the project's actual
  tooling (TypeScript strictness, framework conventions, existing layering).

## Blueprints

Review sequence:

1. Read the relevant file(s); understand the surrounding architecture; see how
   the code is called/used; inspect related modules that affect correctness;
   identify existing conventions; understand dependencies; then review.
2. Work through: What does this code do? → Why does it exist? → How is it
   called? → What data flows through it? → What assumptions does it make? →
   What can go wrong? → Can it be improved?
3. Run the review dimensions against the code:
   - **Correctness** — logic errors, edge cases, race conditions, async bugs,
     incorrect state/error handling, data consistency.
   - **Security** — injection, XSS, CSRF, SSRF, auth bypass, IDOR, sensitive
     data exposure, hardcoded secrets, unsafe uploads, command injection, path
     traversal, missing rate limiting, unsafe dependencies. For each:
     `Vulnerability → Attack/Failure scenario → Impact → Fix`.
   - **Error handling** — swallowed errors, generic errors, wrong status codes,
     leaking internals, missing logging, inappropriate retries/retry storms,
     missing timeouts/fallbacks.
   - **Async/concurrency (JS/Node especially)** — unnecessary sequential
     awaits, missing awaits, unhandled promises, missed `Promise.all`, race
     conditions, shared mutable state, event-loop blocking.
   - **Database code** — query correctness, indexes, N+1s, transactions,
     connection management, pagination, SQL injection, migrations. Ask what
     query an ORM abstraction is actually producing when it matters.
   - **API code** — method usage, status codes, validation, authN/authZ,
     structure, pagination, error responses, idempotency, rate limiting,
     versioning.
   - **Frontend (React/Next.js)** — component responsibilities, unnecessary
     re-renders, state management, server/client boundaries, hooks/effects,
     data fetching/caching, loading/error states, accessibility, bundle size.
   - **TypeScript** — unnecessary `any`, incorrect assertions,
     weak/duplicated types, unsafe handling of external data.
   - **Testability** — what behavior actually needs testing, unit vs
     integration vs E2E, focused on real risk.
4. Detect over-engineering (unnecessary classes/interfaces, excessive patterns,
   unneeded abstractions) — ask "Does this abstraction solve a real problem
   right now?"
5. Detect under-engineering (payments without idempotency, auth without rate
   limiting, external calls without timeouts, unvalidated input, missing
   error handling, missing authz checks) — explain the concrete production risk.
6. For every substantive problem, walk:
   `Current Approach → Problem → Why It Matters → Better Approach → Why It's
   Better → Trade-Off`. Always include the trade-off.
7. Provide the What / Why / When for every meaningful finding, and show code
   sparingly as Current → Improved with the smallest meaningful diff.

## Solutions

Expected output: a structured review. For a substantial review:

```
# Code Review
## Overall Assessment
## What's Good
## Critical Issues
## Important Issues
## Improvements
## Better Approach
## Example Refactor
## Trade-Offs
## What I Would Keep
## What You Should Learn From This
```

For a small snippet or quick question, skip the structure — give a short,
direct answer.

Acceptance criteria:

- Every finding tagged honestly: 🔴 CRITICAL / 🟠 HIGH / 🟡 MEDIUM /
  🔵 LOW / 💡 SUGGESTION.
- Each substantive finding explains Why and When, with the trade-off stated.
- In refactoring-mentor mode, an incremental plan
  (`Small improvement → Run tests → Verify → Next improvement`) is offered
  rather than one large risky rewrite.
- The transferable principle is named, not just the local fix.

## Runtime Constraints and Boundary Checks

- **NEVER**: review a snippet blind when the repo is available; recommend an
  architecture change just because it's fashionable; claim performance issues
  without identifying why (`Bottleneck → Evidence → Measurement →
  Optimization`); nitpick.
- **STOP AND ASK when**: the code's behavior, purpose, or surrounding
  architecture is unclear and can't be determined from the repo.
- **STOP AND FLAG**: anything in the review priority chain that affects
  correctness or security before style.

## Interaction With Other Skills

- **senior-engineer-mentor**: pre-code design/architecture decisions live
  there; this skill covers code that already exists. When a conversation turns
  into a recurring pattern/judgment question, defer upward to the mentor.
- **engineering-orchestrator**: runs after verifier passes, with the repo
  profile as context so the review isn't blind.
