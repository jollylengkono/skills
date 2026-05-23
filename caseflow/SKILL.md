---
name: caseflow
description: Use when starting, organizing, continuing, or closing Oracle customer work cases that need persistent case memory, customer/project folder structure, product skill routing, related-case lookup, or closure summarization.
---

# Caseflow

Caseflow is the intake and memory workflow for Oracle Advanced Services engagements. It creates a consistent customer/project/case workspace, maintains local and central memory, routes to existing Oracle product skills, and links related cases.

## How to Use This Skill

1. Use Caseflow before creating or continuing an Oracle customer work case.
2. Complete the required intake fields before creating folders or memory files.
3. Create case-local memory from `templates/case.md`.
4. Create or update central memory under `<root_workspace_path>/.codex/memories/`.
5. Route to the product skill listed in `Product Routing`; if no product skill exists, follow the missing-skill fallback.
6. Update local and central memory when the case changes or closes.

## Directory Structure

```text
caseflow/
├── SKILL.md
└── templates/
    ├── active-cases.md
    ├── case.md
    ├── customer.md
    ├── pattern.md
    ├── product.md
    ├── project.md
    └── workspace-index.md
```

## Required Inputs

Ask the engineer for these values before creating files:

1. Full root workspace path.
2. Customer name.
3. Project name.
4. Task category: `installation-configuration`, `troubleshooting`, `performance-tuning`, `patching`, or `upgrade`.
5. Oracle product.
6. Short case name, one to three words.

## Product Routing

Use these product abbreviations in case folder names and route to the matching local skill path when available:

| Product | Abbreviation | Local Skill Route |
| --- | --- | --- |
| Database | `db` | `db/SKILL.md` |
| ORDS | `ords` | `db/SKILL.md`, then `db/ords/` |
| APEX | `apex` | `apex/SKILL.md` |
| OCI | `oci` | `oci/SKILL.md` |
| OEM | `oem` | `oem/SKILL.md` |
| GraalVM | `graal` | `graal/SKILL.md` |
| WebLogic | `wls` | `fusion/SKILL.md`, then `fusion/weblogic/` |
| Oracle HTTP Server | `ohs` | `fusion/SKILL.md`, then `fusion/ohs/` |
| SOA Suite | `soa` | `fusion/SKILL.md`, then `fusion/soa/` |
| Oracle Service Bus | `osb` | `fusion/SKILL.md`, then `fusion/soa/` |
| GoldenGate | `gg` | `fusion/SKILL.md`, then `fusion/goldengate/` |
| Java/JVM | `jvm` | `fusion/SKILL.md`, then `fusion/java/` |

For Fusion and OEM products, use the task-category file when it exists:

| Task Category | File |
| --- | --- |
| `installation-configuration` | `installation-and-configuration.md` |
| `troubleshooting` | `troubleshooting.md` |
| `performance-tuning` | `performance-tuning.md` |
| `patching` | `patching-and-upgrade.md` |
| `upgrade` | `patching-and-upgrade.md` |

For `db`, `oci`, `apex`, and `graal`, open the domain `SKILL.md` first and follow that skill's routing table.

## Directory Resolution

1. Confirm the root workspace path exists.
2. Search one level below root for a matching customer directory.
3. If exactly one likely customer match exists, use it.
4. If multiple likely customer matches exist, ask the engineer to choose.
5. If no customer match exists, ask before creating a new customer directory.
6. Repeat the same resolution under the customer directory for the project directory.

## Case Structure

Create the case folder as:

```text
<root_workspace_path>/<customer>/<project>/<product_abbrev>_<short_case_name>_<yyyymmdd>/
```

Create these entries inside it:

```text
case.md
Reference/
Logs/
Docs/
Screenshots/
Scripts/
Output/
```

Use underscores in generated case folder names. Lowercase product abbreviation and short case name. Preserve existing customer and project directory spelling.

## Central Memory

Maintain central memory under the engineer-specified root workspace path. Do not write central memory relative to the current shell directory unless it is the selected root workspace.

```text
<root_workspace_path>/.codex/memories/
  workspace-index.md
  active-cases.md
  customers/<customer>.md
  projects/<customer>/<project>.md
  products/<product>.md
  patterns/<task_category>.md
```

Case-local memory records what happened. Central memory records reusable knowledge.

When a case starts, create missing central memory directories and seed missing files from templates. Update:

- `workspace-index.md` with the root workspace, customer, project, and product coverage.
- `active-cases.md` with the new case path and status.
- `customers/<customer>.md` with the project and case link.
- `projects/<customer>/<project>.md` with product, environment if known, and case link.
- `products/<product>.md` with the case link and reusable product context.
- `patterns/<task_category>.md` with the case link and category-specific evidence needs.

## Related Cases

Add related case links when there is a useful relationship:

- Same customer.
- Same project.
- Same product.
- Same task category.
- Similar symptom or error.
- Shared environment.
- Follow-up work.
- Reusable fix, checklist, or risk.

Each link must include a short reason. Keep links selective and high-signal.

## Subagent Usage

Use subagents where independent work can run in parallel:

- Scan customer memory and prior customer cases.
- Scan project memory and prior project cases.
- Scan product memory and recurring patterns.
- Identify likely related cases.
- Draft the initial engineer checklist.
- Review logs or references.

Do not use subagents for engineer-input decisions: root path, customer, project, task category, product, or short case name.

## Skill Routing

Always apply `using-superpowers` first. Then route by product and task category.

For product work, invoke the matching local product skill route when available. For troubleshooting, also use `systematic-debugging`. Before claiming completion, use `verification-before-completion`.

If no matching local product skill exists:

1. Keep using `using-superpowers`.
2. Use the closest available domain skill if one fits:
   - Unknown middleware product: `fusion/SKILL.md`.
   - Unknown database-adjacent product: `db/SKILL.md`.
   - Unknown cloud infrastructure product: `oci/SKILL.md`.
3. If no close domain match exists, use general process skills:
   - Troubleshooting: `systematic-debugging`.
   - Planning: `writing-plans`.
   - Implementation: `test-driven-development`.
   - Verification: `verification-before-completion`.
4. Record the missing product skill under a `Skill Gaps` section in `<root_workspace_path>/.codex/memories/workspace-index.md`.
5. Suggest creating or installing a new skill later, but do not block case intake.

Task category behavior:

- `installation-configuration`: capture prerequisites, install media, versions, ports, accounts, backups, rollback plan, and post-install validation.
- `troubleshooting`: use `systematic-debugging`; capture symptoms, timeline, scope, recent changes, logs, diagnostics, hypotheses, tests, and validation.
- `performance-tuning`: capture baseline metrics, workload window, AWR or equivalent reports, JVM or OS metrics where relevant, bottleneck hypothesis, one-change-at-a-time actions, and before/after validation.
- `patching`: capture current inventory, target patch, README requirements, conflict checks, backups, rollback plan, maintenance window, apply steps, and post-patch validation.
- `upgrade`: capture source and target versions, certification checks, compatibility risks, backups, fallback plan, rehearsal notes, cutover steps, and post-upgrade validation.

## Initial Output

After intake and case creation, report:

- Created case path.
- Central memory files touched.
- Product and process skills to use.
- Related cases and relevance notes.
- Initial evidence checklist.
- First recommended engineer actions.
- Open questions before risky operations.

## Key Starting Points

- New case intake: start with `Required Inputs`, then `Directory Resolution`.
- Product routing: use `Product Routing`, then load only the matched product skill files.
- Memory setup: use `Central Memory` and the templates under `caseflow/templates/`.
- Similar prior work: use `Related Cases` and scan customer, project, product, and pattern memory.
- Closure: use `Closure` and update both local case memory and central memory.

## Common Mistakes

- Writing central memory under the shell's current directory instead of `<root_workspace_path>/.codex/memories/`.
- Creating a case directly under the customer directory and skipping the project layer.
- Routing directly to a nested Fusion skill without first checking `fusion/SKILL.md`.
- Linking every same-product case instead of adding only high-signal related cases.
- Storing secrets, passwords, wallets, certificates, or production connection strings in central memory.

## Closure

When the engineer says the case is closed, update:

- Local `case.md`.
- `<root_workspace_path>/.codex/memories/customers/<customer>.md`.
- `<root_workspace_path>/.codex/memories/projects/<customer>/<project>.md`.
- `<root_workspace_path>/.codex/memories/products/<product>.md`.
- `<root_workspace_path>/.codex/memories/patterns/<task_category>.md`.
- `<root_workspace_path>/.codex/memories/active-cases.md`.

Closure summaries must capture symptoms, root cause, fix, validation, reusable commands, risks, related cases, and follow-up actions. Do not store secrets.

## Sources

- `README.md`
- `SKILL_AUTHORING_GUIDE.md`
- `db/SKILL.md`
- `apex/SKILL.md`
- `fusion/SKILL.md`
- `oci/SKILL.md`
- `oem/SKILL.md`
- `graal/SKILL.md`
