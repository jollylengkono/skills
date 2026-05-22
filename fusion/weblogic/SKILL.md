---
name: weblogic
description: Oracle WebLogic Server version mapping, lifecycle-aware guidance, and upgrade planning support. Use when Codex must identify WebLogic release lines, translate version labels (11g/12c/14c/15c), compare source and target versions for migrations, build upgrade checklists from WebLogic 11g through the latest 15.x release, or validate release claims against Oracle documentation.
---

# WebLogic

## Overview

Use this skill to standardize WebLogic version naming and produce migration guidance from 11g through 15c.
Load `references/weblogic-version-matrix.md` whenever the request includes version-specific claims.

## Core Concepts

- Treat family labels and canonical versions separately:
- `11g` maps to `10.3.6`.
- `12c` can mean `12.1.3` or `12.2.1.4`, so require precision.
- `14c` maps to either `14.1.1` or `14.1.2`.
- `15c` maps to `15.1.1` (current latest major line as of 2026-05-20).
- Separate these compatibility dimensions in every answer:
- WebLogic version
- JDK/runtime compatibility
- Database compatibility used by the applications

## Workflow

1. Identify the exact source version string from the user inventory.
2. Normalize the source and target versions with `references/weblogic-version-matrix.md`.
3. Determine whether the migration can be represented as a single hop or phased hops.
4. Return:
- `Current` and release family
- `Target` and release family
- proposed path
- pre-checks and post-checks
- sources used

## Practical Examples

### Example Request 1

`Map 10.3.6 to a modern supported WebLogic release and give me a migration plan.`

Expected response shape:

- `Current: 10.3.6 (11g)`
- `Target: 15.1.1.0.0 (15c)`
- `Path: 10.3.6 -> 12.2.1.4 -> 14.1.2 -> 15.1.1` (adjust if Oracle support policy for the estate requires a different path)
- `Pre-checks` and `Post-checks`
- Oracle source links

### Example Request 2

`We are on 12c. Is direct move to 15c valid?`

Expected handling:

- Ask whether `12c` means `12.1.3` or `12.2.1.4`.
- Avoid assuming direct in-place upgrade support without an Oracle citation.
- Recommend staged path when version ambiguity or tooling constraints exist.

## Best Practices and Common Mistakes

- Always quote exact source and target version numbers, not just marketing labels.
- Always verify claims against Oracle documentation before asserting support status.
- Always treat unknown PSU/BP level as a risk and call it out.
- Do not collapse WebLogic upgrade guidance with database upgrade guidance.
- Do not state a direct upgrade as supported unless the source confirms it.

## Oracle Version Notes (19c vs 26ai)

This skill targets WebLogic versioning; `19c` and `26ai` are Oracle Database release baselines, not WebLogic releases.
When the request combines WebLogic and database planning:

- Use `19c` as the baseline database compatibility assumption unless the user states otherwise.
- Flag `26ai` features as database-layer changes that are separate from WebLogic major-version mapping.
- Keep WebLogic pathing decisions based on WebLogic documentation, then layer database compatibility checks separately.

## Installation, Configuration, Patching, and Upgrade

- `installation-and-configuration.md` — silent install, domain creation (GUI and WLST), JDBC data sources, JMS, Node Manager setup, post-install validation
- `patching-and-upgrade.md` — OPatch/BSU differences, Bundle Patch and PSU application, rolling patch for clusters, in-place vs out-of-place upgrade (12c → 14c), pre/post patch checklists

## Troubleshooting and Performance Tuning

- `troubleshooting.md` — startup failures, deployment errors, JDBC pool issues, stuck threads, OOM diagnosis
- `performance-tuning.md` — JVM heap/GC, Work Manager thread tuning, connection pool sizing, JTA timeouts

## Sources

- `references/weblogic-version-matrix.md`
- https://docs.oracle.com/middleware/11119/wls/index.html
- https://docs.oracle.com/middleware/1213/wls/index.html
- https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/12.2.1.4/index.html
- https://docs.oracle.com/en/middleware/standalone/weblogic-server/14.1.1.0/index.html
- https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.2/index.html
- https://docs.oracle.com/en/middleware/standalone/weblogic-server/15.1.1/wlsig/index.html
