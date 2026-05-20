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
│   ├── agents/
│   └── references/
├── java/
│   ├── SKILL.md
│   ├── agents/
│   └── references/
├── soa/
│   ├── SKILL.md
│   ├── agents/
│   └── references/
└── weblogic/
    ├── SKILL.md
    ├── agents/
    └── references/
```

## Category Routing

| Topic | Directory |
|-------|-----------|
| WebLogic version mapping, upgrade sequencing, 11g/12c/14c/15c translation | `fusion/weblogic/` |
| GoldenGate replication topology, legacy 12c to modern release planning, migration cutover planning, and lag troubleshooting | `fusion/goldengate/` |
| Oracle SOA product-family mapping, 11g-to-latest release planning, and migration inventory analysis | `fusion/soa/` |
| Java runtime version mapping (6 to latest), garbage collection strategy, and JVM troubleshooting | `fusion/java/` |

## Key Starting Points

- `fusion/weblogic/SKILL.md`
- `fusion/weblogic/references/weblogic-version-matrix.md`
- `fusion/goldengate/SKILL.md`
- `fusion/goldengate/references/goldengate-source-map.md`
- `fusion/soa/SKILL.md`
- `fusion/soa/references/soa-version-product-matrix.md`
- `fusion/java/SKILL.md`
- `fusion/java/references/java-version-matrix.md`
- `fusion/java/references/java-gc-reference.md`
- `fusion/java/references/java-troubleshooting-reference.md`

## Common Multi-Step Flows

| Task | Recommended Sequence |
|------|----------------------|
| Plan WebLogic modernization path | `weblogic/SKILL.md` workflow -> version matrix lookup -> migration path with checks |
| Plan GoldenGate rollout | `goldengate/SKILL.md` overview -> source-map validation -> topology + cutover plan |
| Plan SOA modernization from 11g | `soa/SKILL.md` overview -> version/product matrix lookup -> phased migration and verification plan |
| Triage Java runtime incidents | `java/SKILL.md` overview -> troubleshooting reference -> GC reference -> remediation plan |
