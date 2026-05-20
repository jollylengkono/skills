---
name: fusion
description: Oracle Fusion Middleware skill routing and domain navigation. Use when tasks involve Fusion-domain technologies and you need to route to the correct Fusion skill, including WebLogic version planning and Oracle GoldenGate replication guidance.
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

## Key Starting Points

- `fusion/weblogic/SKILL.md`
- `fusion/weblogic/references/weblogic-version-matrix.md`
- `fusion/goldengate/SKILL.md`
- `fusion/goldengate/references/goldengate-source-map.md`

## Common Multi-Step Flows

| Task | Recommended Sequence |
|------|----------------------|
| Plan WebLogic modernization path | `weblogic/SKILL.md` workflow -> version matrix lookup -> migration path with checks |
| Plan GoldenGate rollout | `goldengate/SKILL.md` overview -> source-map validation -> topology + cutover plan |
