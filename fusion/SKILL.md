---
name: fusion
description: Oracle Fusion Middleware skill routing and domain navigation. Use when tasks involve Fusion-domain technologies and you need to route to the correct Fusion skill, including WebLogic version planning, Oracle GoldenGate replication guidance, Oracle SOA product/version mapping, Java runtime GC/troubleshooting playbooks, and Oracle HTTP Server (OHS) proxy and web-tier configuration.
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
│   ├── installation-and-configuration.md
│   ├── patching-and-upgrade.md
│   ├── troubleshooting.md
│   ├── performance-tuning.md
│   └── references/
├── java/
│   ├── SKILL.md
│   ├── installation-and-configuration.md
│   ├── patching-and-upgrade.md
│   ├── troubleshooting.md
│   ├── performance-tuning.md
│   └── references/
├── ohs/
│   ├── SKILL.md
│   ├── installation-and-configuration.md
│   ├── patching-and-upgrade.md
│   ├── troubleshooting.md
│   ├── performance-tuning.md
│   └── references/
│       └── certification-matrix.md
├── soa/
│   ├── SKILL.md
│   ├── installation-and-configuration.md
│   ├── patching-and-upgrade.md
│   ├── troubleshooting.md
│   ├── performance-tuning.md
│   └── references/
└── weblogic/
    ├── SKILL.md
    ├── installation-and-configuration.md
    ├── patching-and-upgrade.md
    ├── troubleshooting.md
    ├── performance-tuning.md
    └── references/
```

## Category Routing

| Topic | File |
|-------|------|
| WebLogic version mapping, upgrade sequencing, 11g/12c/14c/15c translation | `fusion/weblogic/SKILL.md` |
| WebLogic installation, silent install, domain creation, JDBC, Node Manager, post-install validation | `fusion/weblogic/installation-and-configuration.md` |
| WebLogic patching (OPatch/BSU), Bundle Patch application, rolling patch, in-place/out-of-place upgrade | `fusion/weblogic/patching-and-upgrade.md` |
| WebLogic startup failures, deployment errors, JDBC pool issues, stuck threads, OOM diagnosis | `fusion/weblogic/troubleshooting.md` |
| WebLogic JVM heap/GC, Work Manager, connection pool, and JTA timeout tuning | `fusion/weblogic/performance-tuning.md` |
| GoldenGate replication topology, legacy 12c to modern release planning, and migration cutover planning | `fusion/goldengate/SKILL.md` |
| GoldenGate installation, classic vs microservices, Extract/Replicat creation, supplemental logging | `fusion/goldengate/installation-and-configuration.md` |
| GoldenGate patching/upgrade, 12c to 21c upgrade, rolling upgrade, pre/post checklists | `fusion/goldengate/patching-and-upgrade.md` |
| GoldenGate Extract ABENDs, Replicat lag, trail file issues, GGSCI diagnostic commands | `fusion/goldengate/troubleshooting.md` |
| GoldenGate Extract and Replicat throughput tuning, trail sizing, TCP buffer tuning | `fusion/goldengate/performance-tuning.md` |
| Oracle SOA product-family mapping, 11g-to-latest release planning, and migration inventory analysis | `fusion/soa/SKILL.md` |
| SOA Suite installation, RCU schema, domain creation, OSB config, adapter configuration | `fusion/soa/installation-and-configuration.md` |
| SOA Suite patching, Bundle Patch, RCU schema upgrade, 11g-to-12c upgrade path | `fusion/soa/patching-and-upgrade.md` |
| SOA BPEL instance failures, Mediator routing errors, OSB issues, JMS/adapter and MDS faults | `fusion/soa/troubleshooting.md` |
| SOA BPEL dehydration, Mediator threading, OSB pipeline, SOAINFRA JDBC, and purge tuning | `fusion/soa/performance-tuning.md` |
| Java runtime version mapping (6 to latest), garbage collection strategy, and JVM troubleshooting | `fusion/java/SKILL.md` |
| Java installation, JAVA_HOME config, JVM flag baseline for middleware, version requirements per product | `fusion/java/installation-and-configuration.md` |
| Java CPU patching, LTS-to-LTS upgrade (8→11→17→21→25), module system impact, middleware re-validation | `fusion/java/patching-and-upgrade.md` |
| Java startup failures, OOM classification, classloading issues in middleware context | `fusion/java/troubleshooting.md` |
| Java heap sizing, GC selection, JVM flag hygiene, JFR profiling for middleware workloads | `fusion/java/performance-tuning.md` |
| OHS installation, standalone vs collocated domain, SSL/TLS, mod_weblogic proxy, virtual hosts | `fusion/ohs/installation-and-configuration.md` |
| OHS patching, Bundle Patch application, upgrade path, rolling patch for HA | `fusion/ohs/patching-and-upgrade.md` |
| OHS process failures, opmnctl status, startup errors, SSL handshake failures, 502/503 proxy errors | `fusion/ohs/troubleshooting.md` |
| OHS MPM tuning, KeepAlive settings, mod_weblogic connection pool, SSL session cache | `fusion/ohs/performance-tuning.md` |
| OHS supported OS, version line, JDK requirements, WebLogic integration compatibility | `fusion/ohs/references/certification-matrix.md` |

## Key Starting Points

- `fusion/weblogic/SKILL.md`
- `fusion/weblogic/installation-and-configuration.md`
- `fusion/weblogic/patching-and-upgrade.md`
- `fusion/weblogic/troubleshooting.md`
- `fusion/weblogic/performance-tuning.md`
- `fusion/weblogic/references/weblogic-version-matrix.md`
- `fusion/goldengate/SKILL.md`
- `fusion/goldengate/installation-and-configuration.md`
- `fusion/goldengate/patching-and-upgrade.md`
- `fusion/goldengate/troubleshooting.md`
- `fusion/goldengate/performance-tuning.md`
- `fusion/goldengate/references/goldengate-source-map.md`
- `fusion/soa/SKILL.md`
- `fusion/soa/installation-and-configuration.md`
- `fusion/soa/patching-and-upgrade.md`
- `fusion/soa/troubleshooting.md`
- `fusion/soa/performance-tuning.md`
- `fusion/soa/references/soa-version-product-matrix.md`
- `fusion/java/SKILL.md`
- `fusion/java/installation-and-configuration.md`
- `fusion/java/patching-and-upgrade.md`
- `fusion/java/troubleshooting.md`
- `fusion/java/performance-tuning.md`
- `fusion/java/references/java-version-matrix.md`
- `fusion/java/references/java-gc-reference.md`
- `fusion/java/references/java-troubleshooting-reference.md`
- `fusion/ohs/SKILL.md`
- `fusion/ohs/installation-and-configuration.md`
- `fusion/ohs/patching-and-upgrade.md`
- `fusion/ohs/troubleshooting.md`
- `fusion/ohs/performance-tuning.md`
- `fusion/ohs/references/certification-matrix.md`

## Common Multi-Step Flows

| Task | Recommended Sequence |
|------|----------------------|
| Deploy WebLogic from scratch | `weblogic/installation-and-configuration.md` → `weblogic/SKILL.md` version check |
| Plan WebLogic modernization path | `weblogic/SKILL.md` workflow → version matrix lookup → `weblogic/patching-and-upgrade.md` |
| Troubleshoot WebLogic runtime issue | `weblogic/troubleshooting.md` triage workflow → version matrix for version-specific notes |
| Tune WebLogic performance | `weblogic/performance-tuning.md` → `java/references/java-gc-reference.md` for GC tuning |
| Deploy GoldenGate from scratch | `goldengate/installation-and-configuration.md` → `goldengate/SKILL.md` topology design |
| Plan GoldenGate rollout | `goldengate/SKILL.md` overview → source-map validation → topology + cutover plan |
| Upgrade GoldenGate | `goldengate/patching-and-upgrade.md` → source-map for certification check |
| Troubleshoot GoldenGate pipeline issue | `goldengate/troubleshooting.md` triage workflow → source-map for release-specific docs |
| Tune GoldenGate throughput or reduce lag | `goldengate/performance-tuning.md` → source-map for release verification |
| Deploy SOA Suite from scratch | `soa/installation-and-configuration.md` → `weblogic/installation-and-configuration.md` for WLS domain base |
| Plan SOA modernization from 11g | `soa/SKILL.md` overview → version/product matrix lookup → `soa/patching-and-upgrade.md` |
| Troubleshoot SOA runtime failures | `soa/troubleshooting.md` triage workflow → `weblogic/troubleshooting.md` for container-level issues |
| Tune SOA Suite performance | `soa/performance-tuning.md` → `java/performance-tuning.md` for JVM layer |
| Upgrade Java for middleware | `java/patching-and-upgrade.md` → `java/references/java-version-matrix.md` for LTS path |
| Triage Java runtime incidents | `java/troubleshooting.md` middleware routing → `java/references/java-troubleshooting-reference.md` commands |
| Tune Java for middleware workloads | `java/performance-tuning.md` → `java/references/java-gc-reference.md` for GC selection |
| Deploy OHS as WebLogic front-end proxy | `ohs/installation-and-configuration.md` → `weblogic/installation-and-configuration.md` for WLS domain pairing |
| Diagnose OHS 502 Bad Gateway | `ohs/troubleshooting.md` triage → mod_weblogic log review → `ohs/installation-and-configuration.md` proxy config check |
| Apply OHS Bundle Patch | `ohs/patching-and-upgrade.md` pre-patch checks → patch → restart → validate |
| Tune OHS for high-traffic proxy workload | `ohs/troubleshooting.md` → `ohs/performance-tuning.md` |
