---
name: universal-package-updater
description: >-
  AI-powered dependency manager and package upgrader. Activates whenever a user
  wants to check for outdated dependencies, update packages, upgrade libraries,
  audit vulnerabilities, manage package versions, or perform any
  package/dependency management task in this project. Triggers on keywords like
  "update packages", "upgrade dependencies", "outdated libraries", "npm update",
  "check vulnerabilities", "dependency audit", "bump versions", "package
  manager", "lock file", "breaking changes", "safe upgrade". Primary ecosystem
  is Node.js/TypeScript (npm/pnpm/yarn/bun), with a support matrix for Python,
  Rust, Go, Java, PHP, Ruby, .NET, Flutter/Dart, and other ecosystems.
---

# Purpose

Scan, analyse, and upgrade dependencies safely, intelligently, and completely —
every time. Never apply a change without classifying its risk, backing up
state, and verifying nothing broke.

## Direction

Goal state: dependencies current within an explicitly agreed risk threshold,
with a risk-assessed report, backup/rollback safety net, and a verified
pass/fail test delta after the update.

Constraints:

- Never assume the package manager — determine it from the lockfile
  (`package-lock.json` → npm, `pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn,
  `bun.lockb` → bun).
- Classify every update before applying it:
  🔒 SECURITY (any version) → update immediately; 🩹 PATCH (1.2.3 → 1.2.x) →
  auto-update (safe); ✨ MINOR (1.2.3 → 1.x.0) → auto-update (usually safe);
  💥 MAJOR (1.2.3 → x.0.0) → interactive with AI guidance; 🪦 DEPRECATED and
  🚫 ABANDONED → flag with replacement.
- Never change the package manager (npm → pnpm) or bump a MAJOR silently.
- Monorepos are scanned per-workspace and aggregated, not assumed uniform.

## Blueprints

Deterministic sequence:

```
DETECT → SCAN → ANALYSE → REPORT → UPDATE → VERIFY
```

1. **DETECT** — identify project type(s) and package manager(s). Detection
   priority: `package.json` (+lockfile) → `pyproject.toml` → `requirements.txt`
   → `Cargo.toml` → `go.mod` → `pom.xml` → `build.gradle(.kts)` →
   `composer.json` → `Gemfile` → `*.csproj` → `pubspec.yaml`. If multiple
   manifests exist in subdirectories, treat as a monorepo.
2. **SCAN** — list all current dependencies with installed vs latest versions.
   Node.js/TypeScript commands: `npm outdated` / `pnpm outdated` / `yarn
   outdated`; security posture via `npm audit` / `pnpm audit` / `yarn audit`.
   Other ecosystems use their equivalent (see Support Matrix).
3. **ANALYSE** — classify each update (SECURITY/PATCH/MINOR/MAJOR/DEPRECATED/
   ABANDONED). For every MAJOR: describe breaking changes in plain language,
   list renamed/removed APIs the project may be using, provide a step-by-step
   migration guide, estimate effort (LOW/MEDIUM/HIGH/VERY HIGH). Always flag
   CVEs and advisories first (CVE number, CVSS score, affected range, patched
   version). Context-aware risk: devDependency vs production dependency,
   transitive vs direct, download count/maintenance status, how recently
   published.
4. **REPORT** — produce the risk-assessed report in the standard format
   (SECURITY / PATCH / MINOR / MAJOR / DEPRECATED sections, summary line,
   risk score, estimated update time).
5. **UPDATE** — apply safe updates (patch + minor) automatically; enter
   interactive mode for every MAJOR: what changed, migration steps, effort
   estimate, then [yes/no/skip].
6. **VERIFY** — run the project's own tests/lint/build before and after, report
   the pass/fail delta. On post-update failure: restore from backup and
   identify the offending package (binary-search approach).

Safety mechanisms, before ANY update:

1. Backup manifest + lock files to `.pkg-backup-<timestamp>/`.
2. Check git status — warn if there are uncommitted changes.
3. Run existing tests before the update.
4. Run existing tests again after — report the delta.
5. If tests fail post-update: restore from backup and report which package
   caused the failure.

Node.js/TypeScript command reference:

```bash
npm outdated / pnpm outdated / yarn outdated      # scan
npm update / pnpm update / yarn upgrade           # patch+minor only
npx npm-check-updates -u --target minor; npm install   # minor within range
npm audit / pnpm audit / yarn audit; <audit> fix  # security
```

## Solutions

Expected output: the standard report plus any updates applied and the verified
test delta.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📦 UNIVERSAL PACKAGE UPDATER — SCAN REPORT
Project: <name>  |  Ecosystem: <ecosystem>  |  Date: <date>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🔒 SECURITY UPDATES (Act Immediately)
  ✗ lodash  3.10.1 → 4.17.21  CVE-2021-23337  CVSS:7.2

🩹 PATCH UPDATES (Safe to Apply)
  • express 4.18.0 → 4.18.3

✨ MINOR UPDATES (Likely Safe)
  • axios   1.3.0  → 1.7.2

💥 MAJOR UPDATES (Requires Review)
  ⚠ react  17.0.2 → 18.3.1  Breaking: concurrent mode, new root API

🪦 DEPRECATED PACKAGES
  ✗ request → Replace with: axios or node-fetch

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Summary: 14 packages scanned | 2 security | 2 patch | 2 minor | 2 major | 1 deprecated
Risk Score: MEDIUM  |  Estimated Update Time: ~45 minutes
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Acceptance criteria:

- Every update classified and every MAJOR explicitly approved before applying.
- Backup exists and restore path was available.
- Tests run before and after; pass/fail delta reported; failures rolled back
  with the offending package identified.
- No package manager changed without a task-tied reason.

## Runtime Constraints and Boundary Checks

| Error | Response |
|---|---|
| Package manager not installed | Print install instructions, abort gracefully |
| Network timeout | Retry 3x with exponential backoff |
| Conflicting peer dependencies | Show conflict tree, suggest resolution |
| Lock file out of sync | Run sync command first, then re-scan |
| Tests fail after update | Auto-rollback, identify offending package |
| No manifest found | Ask user for project root path |
| Permission denied | Suggest activation of virtual env / appropriate privileges |

- **NEVER**: bump MAJOR versions without interactive approval; silently switch
  package managers; apply updates without backup + test-before/test-after;
  auto-apply on a repo with uncommitted changes without warning.
- **STOP AND ASK when**: a MAJOR update's migration effort is HIGH or higher,
  a package is abandoned (flag and suggest alternative), or the repo's
  test/build command can't be determined.
- **STOP AND FLAG**: deprecated packages, abandoned packages, security
  advisories, lockfile inconsistencies (multiple lockfiles present).

## Support Matrix (secondary ecosystems)

| Language | Package Manager | Manifest | Lock file |
|---|---|---|---|
| Python | pip / Poetry / uv / Pipenv | requirements.txt / pyproject.toml / Pipfile | poetry.lock / uv.lock / Pipfile.lock |
| Rust | Cargo | Cargo.toml | Cargo.lock |
| Go | go mod | go.mod | go.sum |
| Java | Maven / Gradle | pom.xml / build.gradle(.kts) | gradle.lockfile |
| PHP | Composer | composer.json | composer.lock |
| Ruby | Bundler | Gemfile | Gemfile.lock |
| .NET | NuGet | *.csproj / packages.config | packages.lock.json |
| Flutter/Dart | pub | pubspec.yaml | pubspec.lock |
