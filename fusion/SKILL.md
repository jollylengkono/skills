---
name: fusion
description: Oracle Fusion Middleware skill routing and domain navigation. Use when tasks involve Fusion-domain technologies and you need to route to the correct Fusion skill, including WebLogic version planning, Oracle GoldenGate replication guidance, Oracle SOA product/version mapping, and Java runtime GC/troubleshooting playbooks.
---

# Oracle Fusion Skills

This domain contains Oracle Fusion Middleware skills. Use the routing table to select the narrowest skill that matches the task.

## How to Use This Domain

1. Identify the middleware product and task type.
2. Open only the matched skill files.
3. Prefer official Oracle sources for version and support claims.

## Directory Structure

```text
fusion/
├── SKILL.md
├── goldengate/
│   ├── SKILL.md
│   ├── troubleshooting.md
│   ├── performance-tuning.md
│   └── references/
├── java/
│   ├── SKILL.md
│   ├── troubleshooting.md
│   ├── performance-tuning.md
│   └── references/
├── soa/
│   ├── SKILL.md
│   ├── troubleshooting.md
│   ├── performance-tuning.md
│   └── references/
└── weblogic/
    ├── SKILL.md
    ├── troubleshooting.md
    ├── performance-tuning.md
    └── references/
```

## Category Routing

| Topic | File |
|-------|------|
| WebLogic version mapping, upgrade sequencing, 11g/12c/14c/15c translation | `fusion/weblogic/SKILL.md` |
| WebLogic startup failures, deployment errors, JDBC pool issues, stuck threads, OOM diagnosis | `fusion/weblogic/troubleshooting.md` |
| WebLogic JVM heap/GC, Work Manager, connection pool, and JTA timeout tuning | `fusion/weblogic/performance-tuning.md` |
| GoldenGate replication topology, legacy 12c to modern release planning, and migration cutover planning | `fusion/goldengate/SKILL.md` |
| GoldenGate Extract ABENDs, Replicat lag, trail file issues, GGSCI diagnostic commands | `fusion/goldengate/troubleshooting.md` |
| GoldenGate Extract and Replicat throughput tuning, trail sizing, TCP buffer tuning | `fusion/goldengate/performance-tuning.md` |
| Oracle SOA product-family mapping, 11g-to-latest release planning, and migration inventory analysis | `fusion/soa/SKILL.md` |
| SOA BPEL instance failures, Mediator routing errors, OSB issues, JMS/adapter and MDS faults | `fusion/soa/troubleshooting.md` |
| SOA BPEL dehydration, Mediator threading, OSB pipeline, SOAINFRA JDBC, and purge tuning | `fusion/soa/performance-tuning.md` |
| Java runtime version mapping (6 to latest), garbage collection strategy, and JVM troubleshooting | `fusion/java/SKILL.md` |
| Java startup failures, OOM classification, classloading issues in middleware context | `fusion/java/troubleshooting.md` |
| Java heap sizing, GC selection, JVM flag hygiene, JFR profiling for middleware workloads | `fusion/java/performance-tuning.md` |

## Key Starting Points

- `fusion/weblogic/SKILL.md`
- `fusion/weblogic/troubleshooting.md`
- `fusion/weblogic/performance-tuning.md`
- `fusion/weblogic/references/weblogic-version-matrix.md`
- `fusion/goldengate/SKILL.md`
- `fusion/goldengate/troubleshooting.md`
- `fusion/goldengate/performance-tuning.md`
- `fusion/goldengate/references/goldengate-source-map.md`
- `fusion/soa/SKILL.md`
- `fusion/soa/troubleshooting.md`
- `fusion/soa/performance-tuning.md`
- `fusion/soa/references/soa-version-product-matrix.md`
- `fusion/java/SKILL.md`
- `fusion/java/troubleshooting.md`
- `fusion/java/performance-tuning.md`
- `fusion/java/references/java-version-matrix.md`
- `fusion/java/references/java-gc-reference.md`
- `fusion/java/references/java-troubleshooting-reference.md`

## Common Multi-Step Flows

| Task | Recommended Sequence |
|------|----------------------|
| Plan WebLogic modernization path | `weblogic/SKILL.md` workflow -> version matrix lookup -> migration path with checks |
| Troubleshoot WebLogic runtime issue | `weblogic/troubleshooting.md` triage workflow -> version matrix for version-specific notes |
| Tune WebLogic performance | `weblogic/performance-tuning.md` -> `java/references/java-gc-reference.md` for GC tuning |
| Plan GoldenGate rollout | `goldengate/SKILL.md` overview -> source-map validation -> topology + cutover plan |
| Troubleshoot GoldenGate pipeline issue | `goldengate/troubleshooting.md` triage workflow -> source-map for release-specific docs |
| Tune GoldenGate throughput or reduce lag | `goldengate/performance-tuning.md` -> source-map for release verification |
| Plan SOA modernization from 11g | `soa/SKILL.md` overview -> version/product matrix lookup -> phased migration and verification plan |
| Troubleshoot SOA runtime failures | `soa/troubleshooting.md` triage workflow -> `weblogic/troubleshooting.md` for container-level issues |
| Tune SOA Suite performance | `soa/performance-tuning.md` -> `java/performance-tuning.md` for JVM layer |
| Triage Java runtime incidents | `java/troubleshooting.md` middleware routing -> `java/references/java-troubleshooting-reference.md` commands |
| Tune Java for middleware workloads | `java/performance-tuning.md` -> `java/references/java-gc-reference.md` for GC selection |
