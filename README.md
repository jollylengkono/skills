# Oracle Skills

Oracle Skills is a collection of practical, installable skills for working with Oracle technologies.

The goal is to give developers and agents a single place to find source-backed Oracle guidance across Oracle Database, Oracle Cloud Infrastructure, GraalVM, Oracle Fusion, Oracle APEX, and future domains.

## Installation

Install a domain by appending the root-level domain directory to the repository name:

```bash
npx skills add oracle/skills/db
npx skills add oracle/skills/oci
npx skills add oracle/skills/graal
npx skills add oracle/skills/caseflow
...
```

### Install in Claude Code

This repository also ships as a Claude Code plugin marketplace (`.claude-plugin/marketplace.json`), where each domain folder (`apex`, `db`, `fusion`, `graal`, `oci`) is published as its own plugin.

Register the marketplace, then install the domain plugins you need:

```bash
# Register this repo as a marketplace
/plugin marketplace add oracle/skills

# Install one or more domain plugins
/plugin install db@oracle-skills
/plugin install graal@oracle-skills
```

Already cloned the repo locally? Point the marketplace at the local path instead:

```bash
/plugin marketplace add ./
```

Browse and toggle installed plugins anytime with `/plugin`. Enabled plugins are tracked in `.claude/settings.json` under `enabledPlugins`.

## Repository Goals

- Provide Oracle-wide skills in one repository.
- Define domain entry points that help developers and agents route to the right topic quickly.
- Keep each skill practical, source-backed, and easy to consume on demand.
- Allow each domain to evolve its own taxonomy without breaking repo-wide consistency.

## Domains

- `db/` is the active Oracle Database domain and includes database, ORDS, SQLcl, framework, container, and agent workflow skills.
- `oci/` contains Oracle Cloud Infrastructure skills for landing-zone architecture, IAM/security guardrails, and networking operations, plus OCI Kubernetes Engine (OKE) cluster design and troubleshooting and Enterprise AI guidance for OCI Generative AI, agents, RAG, governance, model endpoints, and integrations.
- `fusion/` contains Oracle Fusion Middleware skills — WebLogic, GoldenGate, SOA, Java, and Oracle HTTP Server (OHS) — each sub-domain has installation, patching/upgrade, troubleshooting, and performance tuning files.
- `apex/` contains Oracle APEX skills, including the APEXLang sub-domain for structured APEX application generation.
- `oem/` contains Oracle Enterprise Manager 13c skills covering installation, patching and upgrade, troubleshooting, performance tuning, and certification matrix.
- `graal/` contains GraalVM skills, starting with Native Image.
- `caseflow/` contains the Oracle work-case intake workflow for customer/project/case structure, persistent memory, product skill routing, and cross-case links.

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
├── caseflow/
│   ├── SKILL.md
│   └── templates/
│       ├── active-cases.md
│       ├── case.md
│       ├── customer.md
│       ├── pattern.md
│       ├── product.md
│       ├── project.md
│       └── workspace-index.md
├── oci/
│   ├── SKILL.md
│   ├── enterprise-ai/
│   │   ├── SKILL.md
│   │   ├── models/
│   │   ├── agent-workflows/
│   │   ├── governance/
│   │   ├── data/
│   │   ├── cost/
│   │   └── integrations/
│   ├── oke/
│   │   ├── cluster-design.md
│   │   ├── troubleshooting.md
│   │   ├── gva-node-pools.md
│   │   ├── multus-multihome.md
│   │   ├── skills/
│   │   ├── scripts/
│   │   ├── agents/
│   │   ├── shared/
│   │   ├── examples/
│   │   └── tests/
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

## Sources

- https://docs.oracle.com/en-us/iaas/Content/ContEng/home.htm
- https://www.graalvm.org/latest/reference-manual/native-image/
