---
name: ohs
description: Oracle HTTP Server (OHS) 12c skill for installation, configuration, patching, troubleshooting, performance tuning, and certification matrix. Use when tasks involve OHS standalone or collocated domain setup, mod_weblogic proxy configuration, SSL/TLS certificate management, opmnctl/opmn operations, virtual host configuration, or diagnosing OHS startup and proxy errors.
---

# Oracle HTTP Server (OHS)

## Overview

Use this skill for guidance on Oracle HTTP Server 12c — Oracle's web server based on Apache HTTP Server, used as a front-end proxy for Oracle Fusion Middleware (WebLogic, SOA Suite, Oracle Forms, Oracle Reports).

Load the relevant skill file for the task at hand. All major claims must be verified against official Oracle documentation before applying to production.

## How to Use This Domain

1. Identify the task type from the category routing table below.
2. Load the matching skill file.
3. Cross-reference the certification matrix before making version-specific decisions.
4. Validate commands and parameters against the Oracle OHS 12.2.1.x documentation for your exact release.

## Directory Structure

```text
fusion/ohs/
├── SKILL.md
├── installation-and-configuration.md
├── patching-and-upgrade.md
├── troubleshooting.md
├── performance-tuning.md
└── references/
    └── certification-matrix.md
```

## Category Routing

| Topic | File |
|-------|------|
| OHS 12c installation (standalone domain, collocated with WebLogic), silent install, SSL/TLS, mod_weblogic, virtual hosts | `installation-and-configuration.md` |
| OHS patching — OPatch / Bundle Patch application, upgrade path, rolling patch for HA, post-patch validation | `patching-and-upgrade.md` |
| OHS process failures, opmnctl status, startup errors, SSL handshake failures, 502/503 proxy errors, log locations | `troubleshooting.md` |
| MPM tuning, KeepAlive settings, mod_weblogic connection pool, SSL session cache, access log buffering | `performance-tuning.md` |
| Supported OS, OHS version line, JDK requirements, WebLogic integration, browser compatibility | `references/certification-matrix.md` |

## Key Starting Points

- New OHS deployment → `installation-and-configuration.md`
- Applying patches or upgrading OHS → `patching-and-upgrade.md`
- OHS not starting / proxy returning 502 / SSL error → `troubleshooting.md`
- OHS slow or high connection backlog → `performance-tuning.md`
- Version/platform support question → `references/certification-matrix.md`

## Common Multi-Step Flows

| Task | Recommended Sequence |
|------|----------------------|
| Fresh OHS 12c deployment | `certification-matrix.md` (verify platform) → `installation-and-configuration.md` |
| Apply OHS Bundle Patch | `patching-and-upgrade.md` pre-patch checks → patch → restart → validate |
| Diagnose OHS 502 Bad Gateway | `troubleshooting.md` triage → mod_weblogic log review → `installation-and-configuration.md` (proxy config check) |
| OHS high latency / connection saturation | `troubleshooting.md` (rule out OHS process issue) → `performance-tuning.md` |
| Configure SSL termination at OHS | `installation-and-configuration.md` (SSL/TLS section) → `troubleshooting.md` (SSL handshake diagnostics) |

## Oracle Version Notes

- OHS 12.2.1.4 is the current release line as of 2026.
- OHS 12c is built on Apache HTTP Server 2.4.x; configuration directives follow Apache 2.4 semantics.
- OHS is managed via OPMN (Oracle Process Manager and Notification Server); use `opmnctl` for start/stop/status operations.
- OHS can be deployed as a standalone Oracle home or collocated within a WebLogic domain.

## Sources

- OHS 12c documentation: https://docs.oracle.com/en/middleware/fusion-middleware/web-tier/12.2.1.4/
- Oracle Fusion Middleware documentation hub: https://docs.oracle.com/en/middleware/fusion-middleware/
- OHS certification matrix (MOS): https://support.oracle.com/epmos/faces/CertifyHome
