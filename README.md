# Oracle Skills

Oracle Skills is a collection of practical, installable skills for working with Oracle technologies.

The goal is to give developers and agents a single place to find source-backed Oracle guidance across Oracle Database, Oracle Cloud Infrastructure, GraalVM, Oracle Fusion, Oracle APEX, and future domains.

## Installation

Install a domain by appending the root-level domain directory to the repository name:

```bash
npx skills add oracle/skills/db
npx skills add oracle/skills/graal
...
```

## Repository Goals

- Provide Oracle-wide skills in one repository.
- Define domain entry points that help developers and agents route to the right topic quickly.
- Keep each skill practical, source-backed, and easy to consume on demand.
- Allow each domain to evolve its own taxonomy without breaking repo-wide consistency.

## Domains

- `db/` is the active Oracle Database domain and includes database, ORDS, SQLcl, framework, container, and agent workflow skills.
- `oci/` contains Oracle Cloud Infrastructure skills for landing-zone architecture, IAM/security guardrails, and networking operations.
- `fusion/` contains Oracle Fusion Middleware skills — WebLogic, GoldenGate, SOA, Java, and Oracle HTTP Server (OHS) — each sub-domain has installation, patching/upgrade, troubleshooting, and performance tuning files.
- `apex/` contains Oracle APEX skills, including the APEXLang sub-domain for structured APEX application generation.
- `oem/` contains Oracle Enterprise Manager 13c skills covering installation, patching and upgrade, troubleshooting, performance tuning, and certification matrix.
- `graal/` contains GraalVM skills, starting with Native Image.

## Start Here

1. Pick the domain closest to your task.
2. Install that domain skill.
3. Add other domain skills only when needed.

## Repository Layout

```text
.
├── db/
│   ├── SKILL.md
│   ├── admin/
│   ├── agent/
│   ├── appdev/
│   ├── architecture/
│   ├── containers/
│   ├── design/
│   ├── devops/
│   ├── features/
│   ├── frameworks/
│   ├── migrations/
│   ├── monitoring/
│   ├── ords/
│   ├── performance/
│   ├── plsql/
│   ├── security/
│   ├── sql-dev/
│   └── sqlcl/
├── fusion/
│   ├── SKILL.md
│   ├── goldengate/
│   │   ├── SKILL.md
│   │   ├── installation-and-configuration.md
│   │   ├── patching-and-upgrade.md
│   │   ├── troubleshooting.md
│   │   ├── performance-tuning.md
│   │   └── references/
│   ├── java/
│   │   ├── SKILL.md
│   │   ├── installation-and-configuration.md
│   │   ├── patching-and-upgrade.md
│   │   ├── troubleshooting.md
│   │   ├── performance-tuning.md
│   │   └── references/
│   ├── ohs/
│   │   ├── SKILL.md
│   │   ├── installation-and-configuration.md
│   │   ├── patching-and-upgrade.md
│   │   ├── troubleshooting.md
│   │   ├── performance-tuning.md
│   │   └── references/
│   │       └── certification-matrix.md
│   ├── soa/
│   │   ├── SKILL.md
│   │   ├── installation-and-configuration.md
│   │   ├── patching-and-upgrade.md
│   │   ├── troubleshooting.md
│   │   ├── performance-tuning.md
│   │   └── references/
│   └── weblogic/
│       ├── SKILL.md
│       ├── installation-and-configuration.md
│       ├── patching-and-upgrade.md
│       ├── troubleshooting.md
│       ├── performance-tuning.md
│       └── references/
├── apex/
│   ├── SKILL.md
│   └── apexlang/
│       ├── SKILL.md
│       ├── assets/
│       └── references/
├── graal/
│   ├── SKILL.md
│   └── native-image/
│       ├── build-native-image.md
│       ├── native-build-tools.md
│       ├── reachability-metadata.md
│       └── troubleshooting.md
├── oci/
│   ├── SKILL.md
│   └── references/
│       ├── landing-zone-core.md
│       ├── iam-security-guardrails.md
│       └── networking-operations.md
└── oem/
    ├── SKILL.md
    ├── installation-and-configuration.md
    ├── patching-and-upgrade.md
    ├── troubleshooting.md
    ├── performance-tuning.md
    └── references/
        └── certification-matrix.md
```

Each domain has its own `SKILL.md` and any supporting index files it needs.

For a real domain, organize content by category directories and use `SKILL.md` as the table of contents. A domain `SKILL.md` should normally include:

- `## How to Use This Domain`
- `## Directory Structure`
- `## Category Routing`
- `## Key Starting Points`
- `## Common Multi-Step Flows`

For stub domains, keep `SKILL.md` minimal and point users back to this `README.md` and `SKILL_AUTHORING_GUIDE.md`.

## Version Coverage Standard

- Skills that include version-specific behavior must include a section named `## Oracle Version Notes (19c vs 26ai)`.
- Use Oracle Database 19c as the baseline compatibility target unless stated otherwise.
- Explicitly call out features that require newer releases and provide 19c-compatible alternatives where practical.
