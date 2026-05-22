# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Before Writing or Updating Any Skill

Always read `README.md` and `SKILL_AUTHORING_GUIDE.md` first. They define the authoritative authoring standards, domain conventions, and research requirements for this repository.

## What This Repository Is

A collection of installable Oracle technology skills — curated markdown guidance organized by domain. There is no build system, compiled code, or test runner. Work here is entirely authoring and editing markdown files.

Skills are installed by end users via:
```bash
npx skills add oracle/skills/db
npx skills add oracle/skills/oci
# etc.
```

## Domain Layout

Each root-level directory is an installable domain:

| Domain | Coverage |
|--------|----------|
| `db/` | Oracle Database — SQL, PL/SQL, ORDS, SQLcl, admin, security, performance, migrations, agent workflows |
| `oci/` | Oracle Cloud Infrastructure — landing zones, IAM/security guardrails, networking operations |
| `fusion/` | Oracle Fusion Middleware — WebLogic, GoldenGate, SOA, Java runtime; each sub-domain has installation, patching/upgrade, troubleshooting, and performance tuning files |
| `graal/` | GraalVM Native Image — CLI builds, Maven/Gradle Native Build Tools, metadata, troubleshooting |
| `apex/` | Oracle APEX — APEXLang sub-domain for structured APEX application generation |
| `oem/` | Oracle Enterprise Manager 13c — installation, patching and upgrade (merged), troubleshooting, performance tuning, certification matrix |
| `ohs/` | Oracle HTTP Server 12c — installation, patching/upgrade, troubleshooting, performance tuning, certification matrix |

## Entry Point Convention

Every installable domain must have a `SKILL.md` at its root with YAML frontmatter:
```yaml
---
name: <domain-slug>
description: <one-line routing description>
---
```

The `SKILL.md` acts as a table of contents and router. It must contain sections: `## How to Use This Domain`, `## Directory Structure`, `## Category Routing`, `## Key Starting Points`, and `## Common Multi-Step Flows`.

Sub-domains (e.g. `fusion/weblogic/`) follow the same pattern with their own `SKILL.md`.

## Standard File Set Per Sub-Domain

Full domains (fusion sub-domains, oem, ohs) use this standard file set:

- `SKILL.md` — router and table of contents
- `installation-and-configuration.md` — install modes, silent install, post-install validation
- `patching-and-upgrade.md` — patch types, patch application, version upgrade path, rollback
- `troubleshooting.md` — triage workflows, log locations, diagnostic commands
- `performance-tuning.md` — tuning parameters, capacity planning, JVM/GC guidance
- `references/` — certification matrix, version matrices, source maps

Not every domain needs all files. Use the standard set as the target for mature domains.

## Authoring a New Skill File

Preferred structure for any skill file:
1. `## Overview`
2. Problem framing or core concepts
3. Practical examples
4. Best practices and common mistakes
5. `## Oracle Version Notes (19c vs 26ai)` — required when version behavior differs
6. `## Sources`

**All major claims must be backed by official Oracle sources.** Preferred order: `docs.oracle.com` → Oracle-owned repos → Oracle-authored blogs → Oracle LiveLabs. Do not include invented defaults, undocumented env vars, or guessed CLI flags.

Use Oracle Database 19c as the baseline compatibility target unless the domain specifies otherwise. Explicitly call out features requiring newer releases.

## File Naming and Placement

- Lowercase with hyphens: `sqlcl-mcp-server.md`, not `SQLclMCPServer.md`
- One primary topic per file
- Path: `<domain>/<category>/<topic>.md`
- Avoid marketing-language filenames unless that name is the official product term
- Prefer merged files over many narrow files for closely related topics (e.g. `patching-and-upgrade.md` not separate `patching.md` + `upgrade.md`)

## Required Follow-Through When Adding or Moving Skills

1. Update the domain `SKILL.md` (category routing, key starting points, multi-step flows).
2. Update `README.md` if the domain layout or repo navigation changed.
3. Check for stale links across the repo.

## Contributing

All commits to this repo require a `Signed-off-by` line (Oracle Contributor Agreement):
```bash
git commit --signoff
```

The `main` branch requires a PR with at least one approving review, linear history, and no force-pushes.
