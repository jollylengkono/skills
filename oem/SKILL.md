---
name: oem
description: Oracle Enterprise Manager (OEM) 13c skill for installation, configuration, patching, troubleshooting, performance tuning, and certification matrix. Use when tasks involve OEM OMS setup, agent deployment, target management, plug-in patching, repository tuning, emctl/emcli operations, or diagnosing OEM console and agent issues.
---

# Oracle Enterprise Manager (OEM)

## Overview

Use this skill for guidance on Oracle Enterprise Manager 13c — Oracle's unified management platform for Oracle Database, Oracle Middleware, and hybrid cloud environments.

Load the relevant skill file for the task at hand. All major claims must be verified against official Oracle documentation before applying to production.

## How to Use This Domain

1. Identify the task type from the category routing table below.
2. Load the matching skill file.
3. Cross-reference the certification matrix before making version-specific decisions.
4. Validate commands and parameters against the Oracle OEM 13c documentation for your exact release (13.4 or 13.5).

## Directory Structure

```text
oem/
├── SKILL.md
├── installation-and-configuration.md
├── patching.md
├── troubleshooting.md
├── performance-tuning.md
└── references/
    └── certification-matrix.md
```

## Category Routing

| Topic | File |
|-------|------|
| OEM 13c installation (Simple, Advanced, Software Only), post-install config, agent deployment, HA setup | `installation-and-configuration.md` |
| OEM patching — Bundle Patch, Release Update, one-off patches, plug-in and agent mass upgrade | `patching.md` |
| OMS startup failures, agent unreachable, OMR connectivity, console errors, job failures, log locations | `troubleshooting.md` |
| OMS JVM tuning, OMR database tuning, agent collection tuning, WebLogic thread pool, data retention | `performance-tuning.md` |
| Supported OS, DB target versions, JDK, plug-in versions, browser compatibility | `references/certification-matrix.md` |

## Key Starting Points

- New OEM deployment → `installation-and-configuration.md`
- Applying patches or upgrades → `patching.md`
- OEM not working / agent down / console error → `troubleshooting.md`
- OEM slow / OMR bloated / agent overloading DB → `performance-tuning.md`
- Version/platform support question → `references/certification-matrix.md`

## Common Multi-Step Flows

| Task | Recommended Sequence |
|------|----------------------|
| Fresh OEM 13c deployment | `certification-matrix.md` (verify platform) → `installation-and-configuration.md` |
| Apply OEM Bundle Patch | `patching.md` pre-patch checks → patch OMS → patch agents → verify |
| Diagnose OEM agent unreachable | `troubleshooting.md` triage → log review → `emctl` commands |
| OEM console is slow | `troubleshooting.md` (rule out OMS/OMR issue) → `performance-tuning.md` |
| Add new database target | `installation-and-configuration.md` (agent deploy + target discovery) |

## Oracle Version Notes

- OEM 13.5 is the current release line as of 2026.
- OEM 13.4 is the prior release; upgrade to 13.5 is recommended.
- For managed Oracle Database targets: use Oracle Database 19c as the baseline compatibility assumption unless stated otherwise.
- OEM 13c manages both on-premises and Oracle Cloud targets; cloud-specific features require additional configuration.

## Sources

- OEM 13c documentation hub: https://docs.oracle.com/en/enterprise-manager/
- OEM 13.5 Cloud Control documentation: https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/
- OEM certification matrix (MOS): https://support.oracle.com/epmos/faces/CertifyHome
