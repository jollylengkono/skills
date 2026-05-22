---
name: soa
description: Oracle SOA Suite version and product-family mapping guidance for architecture, upgrade, and migration planning. Use when Codex must map Oracle SOA releases from 11g through the latest line, identify which Oracle SOA products are listed per release, validate legacy-to-modern modernization paths, or produce migration checklists for SOA Suite, Service Bus, B2B, BPM, MFT, and related SOA products.
---

# Oracle SOA

## Overview

Use this skill to build source-backed Oracle SOA guidance for version mapping and product coverage planning.
Load `references/soa-version-product-matrix.md` when a request includes release/version claims or asks whether a SOA product belongs to a given release line.

## Core Concepts

- Separate release naming from product availability.
- Normalize user inputs to release lines (`11g`, `12c`, `14c`) before proposing migration paths.
- Treat `14.1.2` as the latest Oracle SOA Suite line as of 2026-05-20.
- Distinguish `listed in docs` from `certified/supported in your estate`; certification checks are still required.
- Treat 11g-era components as legacy modernization candidates unless a user explicitly requires backward compatibility.

## Practical Examples

### Example 1: 11g to Latest SOA Upgrade Planning

User request: `We run SOA 11g with BPEL, Mediator, and OSB. Give a modernization path to latest SOA.`

Response pattern:

1. Identify exact 11g baseline (`11.1.1.x`) and deployed products.
2. Map source products and release line using the matrix.
3. Propose phased target planning (`12c` staging if required by estate constraints, then `14.1.2`).
4. Provide pre-check and post-check gates.

### Example 2: Product Coverage Validation

User request: `Is Managed File Transfer part of Oracle SOA products in 12c and latest?`

Response pattern:

1. Look up product row in the matrix.
2. Return listed availability by release line.
3. Add a note to validate support/certification for exact platform and versions.

### Example 3: Portfolio Inventory Mapping

User request: `Build a table of all Oracle SOA products and where they appear from 11g to latest.`

Response pattern:

1. Use the product coverage matrix directly.
2. Preserve notes for legacy-only or release-specific products.
3. Distinguish missing listing from explicit de-support.

## Best Practices and Common Mistakes

- Always capture exact release numbers before giving migration advice.
- Always cite Oracle documentation links for product/release claims.
- Always separate documentation listing from certification/support decisions.
- Do not assume every 11g product has a direct equivalent in later releases.
- Do not claim a direct in-place upgrade path without a source-backed compatibility check.

## Oracle Version Notes (19c vs 26ai)

This section applies when SOA projects also involve Oracle Database planning.

- Use `19c` as the database baseline assumption unless the user specifies a different baseline.
- For targets that include database `26ai`, require explicit compatibility/certification checks in addition to SOA release mapping.
- Keep SOA release planning and database version planning as separate decision tracks, then reconcile them at cutover design.

## Troubleshooting and Performance Tuning

- `troubleshooting.md` — BPEL instance failures, Mediator routing errors, OSB issues, JMS/adapter faults, MDS metadata problems
- `performance-tuning.md` — BPEL dehydration tuning, Mediator threading, OSB pipeline optimization, SOAINFRA JDBC tuning, instance purging

## Sources

- `references/soa-version-product-matrix.md`
- https://docs.oracle.com/en/middleware/soa-suite/
- https://www.oracle.com/middleware/technologies/soa-suite-11gdocumentation.html
- https://www.oracle.com/middleware/technologies/soa-suite.html
- https://www.oracle.com/middleware/technologies/fusion-certification.html
