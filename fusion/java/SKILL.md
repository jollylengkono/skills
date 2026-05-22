---
name: java
description: Java runtime version mapping, garbage collection guidance, and JVM troubleshooting playbooks for Fusion-domain operations. Use when Codex needs to map Java versions from Java 6 through the latest release, plan LTS-to-LTS upgrades, diagnose GC behavior, troubleshoot JVM startup/runtime failures, or produce Java migration checklists for middleware estates.
---

# Java

## Overview

Use this skill to provide source-backed Java operational guidance across legacy and modern runtimes.
Load the reference files for version timeline, GC strategy, and troubleshooting workflows.

## Core Concepts

- Normalize Java estate inventory first (`java -version`, runtime vendor/distribution, key JVM flags).
- Treat Java 6 and 7 as legacy migration sources.
- Treat Java 26 as the latest feature line and Java 25 as the latest LTS line as of 2026-05-20.
- Prefer LTS-to-LTS upgrade planning unless the user explicitly needs a feature-line target.
- Separate `available in docs` from `supported in your platform estate`.

## Practical Examples

### Example 1: Java 6 to Current LTS Upgrade Path

User request: `We still run Java 6. Give me a safe target path to modern Java.`

Response pattern:

1. Validate source runtime and dependencies.
2. Map release path using `references/java-version-matrix.md`.
3. Propose phased upgrade path (`6 -> 8 -> 11 -> 17 -> 21 -> 25`) with risk gates.
4. Call out removed components and module encapsulation impacts.

### Example 2: GC Collector Migration

User request: `Our old CMS tuning is failing on new JDK. What do we migrate to?`

Response pattern:

1. Identify current and target JDK versions.
2. Check collector availability and defaults using `references/java-gc-reference.md`.
3. Propose collector migration (typically G1, ZGC, or Shenandoah depending on goals).
4. Provide logging/tuning rollout steps and validation checks.

### Example 3: JVM Troubleshooting Runbook

User request: `JVM crashes at startup and sometimes OOMs under load. Give me a triage flow.`

Response pattern:

1. Use `references/java-troubleshooting-reference.md` fast triage workflow.
2. Separate startup failures, memory/GC pressure, thread/CPU issues, and classloading/module issues.
3. Recommend low-risk first diagnostics (`jcmd`, thread dump, heap summary), then deeper artifacts (heap dump/JFR).
4. Return specific next commands and expected evidence.

## Best Practices and Common Mistakes

- Always pin exact Java major/minor line before giving compatibility advice.
- Always cite Oracle/OpenJDK sources for release and feature-removal claims.
- Always start migration planning from a minimal flag set, then tune from evidence.
- Do not carry legacy JVM flags forward without revalidation.
- Do not assume behavior from Java 8 applies unchanged in Java 9+ modular runtimes.

## Troubleshooting and Performance Tuning

- `troubleshooting.md` — middleware-specific startup failures, OOM classification, stuck threads, classloading/module issues across WebLogic, SOA, and GoldenGate
- `performance-tuning.md` — heap sizing, GC selection for middleware workloads, JVM flag hygiene across JDK upgrades, JFR profiling, Metaspace tuning

## References

- Use `references/java-version-matrix.md` for Java 6 to latest mapping and LTS planning.
- Use `references/java-gc-reference.md` for collector availability/defaults/tuning.
- Use `references/java-troubleshooting-reference.md` for incident runbooks and commands.

## Oracle Version Notes (19c vs 26ai)

This section applies when Java planning intersects Oracle Database version planning.

- Use Oracle Database `19c` as the baseline assumption unless user context states otherwise.
- For estates targeting database `26ai`, validate Java runtime compatibility in the middleware/application stack separately from database version decisions.
- Keep Java runtime upgrades and database upgrades as separate tracks, then reconcile at integration and cutover testing.

## Sources

- `references/java-version-matrix.md`
- `references/java-gc-reference.md`
- `references/java-troubleshooting-reference.md`
- https://www.java.com/en/releases/
- https://www.oracle.com/java/technologies/java-se-support-roadmap.html
- https://docs.oracle.com/en/java/javase/26/
