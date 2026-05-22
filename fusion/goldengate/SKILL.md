---
name: goldengate
description: Oracle GoldenGate replication design, implementation, upgrade, and troubleshooting guidance. Use when Codex needs to plan CDC topologies, validate GoldenGate prerequisites, map source and target compatibility across legacy 12c and newer releases, prepare migration cutover steps, troubleshoot Extract/Replicat lag, or compare architecture choices for Oracle and heterogeneous databases.
---

# Oracle GoldenGate

## Overview

Use this skill to build practical, source-backed Oracle GoldenGate guidance for migration, replication, and operational troubleshooting.
Load `references/goldengate-source-map.md` whenever version support, certification, or command behavior must be verified.

## Core Concepts

- Model every flow as three stages: capture, route, and apply.
- Define topology early: unidirectional, active-passive failover, or bidirectional.
- Treat prerequisite validation as mandatory before parameter tuning.
- Validate supplemental logging and primary key coverage.
- Validate source and target version support from certification sources.
- Validate network, time sync, and security baseline.
- Treat GoldenGate `12c` estates as legacy and verify exact release (`12.1.2`, `12.2.0.1`, or `12.3.0.1`) before proposing upgrades.
- Separate logical replication design from cutover execution.

## Practical Examples

### Example 1: Near-Zero Downtime Database Migration

User request: `Plan GoldenGate migration from Oracle Database 19c to Oracle AI Database 26ai with minimal downtime.`

Response pattern:

1. Confirm source and target versions, edition, and platform.
2. Verify GoldenGate release support in certification sources.
3. Produce a phased plan:
- initial load
- CDC catch-up window
- lag threshold and cutover gate
- rollback window
4. Provide validation checks:
- trail growth and delivery health
- Extract/Replicat checkpoint progress
- row-count and reconciliation checks

### Example 2: Replicat Lag Triage

User request: `Replicat lag jumped from seconds to 45 minutes.`

Response pattern:

1. Localize lag location: capture, distribution, or apply.
2. Check for apply-side blockers first (errors, locks, DDL conflicts).
3. Separate throughput bottleneck from data quality issues.
4. Return mitigation options with risk notes and a post-change verification plan.

### Example 3: Heterogeneous Replication Feasibility

User request: `Can we replicate Oracle source to non-Oracle target?`

Response pattern:

1. Confirm exact source and target engines and versions.
2. Check certification matrix and release-specific docs.
3. Return support verdict with cited source links.
4. If unsupported or unclear, provide safest fallback path.

### Example 4: Legacy 12c Upgrade Planning

User request: `We run GoldenGate 12c. Give a safe path to modern GoldenGate for an Oracle 19c/26ai migration.`

Response pattern:

1. Identify exact 12c release line from inventory (`12.1.2`, `12.2.0.1`, or `12.3.0.1`).
2. Validate release support and certification before selecting target GoldenGate release.
3. Recommend staged upgrade plus replication cutover checkpoints.
4. Include rollback triggers and post-cutover validation gates.

## Best Practices and Common Mistakes

- Always cite official Oracle documentation for support claims.
- Always request exact source and target versions before giving a compatibility answer.
- Always include a rollback and verification step in migration plans.
- Do not assume a feature is available across all GoldenGate release lines.
- Do not skip supplemental logging and key-column checks.
- Do not present syntax or parameters without a source when version-sensitive.

## Oracle Version Notes (19c vs 26ai)

This section refers to Oracle Database versions often involved in GoldenGate projects.

- Use `19c` as the default baseline when database version is unspecified.
- For `26ai` targets, require an explicit certification check before selecting the GoldenGate release.
- In mixed `19c` to `26ai` migrations, prioritize conservative staged cutover planning and documented fallback.
- If release-specific feature parity is unclear, provide a 19c-compatible operational fallback and cite uncertainty.
- If source systems are still on database `12c`, treat that as a legacy migration path and require explicit compatibility checks before finalizing GoldenGate release selection.

## Installation, Configuration, Patching, and Upgrade

- `installation-and-configuration.md` — classic vs microservices architecture install, GGSCI vs Service Manager, supplemental logging setup, Extract/Replicat creation, trail file config, initial load, connectivity validation
- `patching-and-upgrade.md` — OPatch for classic GoldenGate, upgrade from 12c to 21c, rolling upgrade for zero-downtime, microservices deployment upgrade, pre/post upgrade checklists

## Troubleshooting and Performance Tuning

- `troubleshooting.md` — Extract ABENDs, Replicat lag spikes, trail file issues, discard file triage, GGSCI diagnostic commands
- `performance-tuning.md` — Integrated Extract tuning, Parallel Replicat parallelism, trail sizing, TCP buffer tuning, supplemental logging impact

## Sources

- `references/goldengate-source-map.md`
- https://docs.oracle.com/en/database/goldengate/core/
- https://docs.oracle.com/en/middleware/goldengate/core/12.3.0.1/books.html
- https://docs.oracle.com/en/middleware/goldengate/
- https://docs.oracle.com/en/middleware/goldengate/core/19.1/
- https://docs.oracle.com/en/middleware/goldengate/core/21.3/ggcab/overview-oracle-goldengate.html
- https://docs.oracle.com/en/middleware/goldengate/core/23/coredoc/toc.htm
- https://docs.oracle.com/en/middleware/goldengate/core/21.3/coredoc/install-verify-certification-and-system-requirements.html
- https://www.oracle.com/integration/goldengate/certifications/
