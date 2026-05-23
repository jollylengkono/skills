---
name: caseflow
description: Use when starting, organizing, continuing, or closing Oracle customer work cases that need persistent case memory, customer/project folder structure, product skill routing, related-case lookup, or closure summarization.
---

# Caseflow

Caseflow is the intake and memory workflow for Oracle Advanced Services engagements. It creates a consistent customer/project/case workspace, maintains local and central memory, routes to existing Oracle product skills, and links related cases.

## Required Inputs

Ask the engineer for these values before creating files:

1. Full root workspace path.
2. Customer name.
3. Project name.
4. Task category: `installation-configuration`, `troubleshooting`, `performance-tuning`, `patching`, or `upgrade`.
5. Oracle product.
6. Short case name, one to three words.

## Product Abbreviations

Use these prefixes in case folder names:

| Product | Abbreviation | Product Skill |
| --- | --- | --- |
| Database | `db` | `db` |
| WebLogic | `wls` | `weblogic` |
| Oracle HTTP Server | `ohs` | `ohs` |
| SOA Suite | `soa` | `soa` |
| Oracle Service Bus | `osb` | `soa` |
| GoldenGate | `gg` | `goldengate` |
| APEX | `apex` | `apex` |
| OEM | `oem` | `oem` |
| OCI | `oci` | `oci` |
| Java/JVM | `jvm` | `java` |
| ORDS | `ords` | `apex` |

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

For product work, invoke the matching product skill when available. For troubleshooting, also use `systematic-debugging`. Before claiming completion, use `verification-before-completion`.

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

## Closure

When the engineer says the case is closed, update:

- Local `case.md`.
- `<root_workspace_path>/.codex/memories/customers/<customer>.md`.
- `<root_workspace_path>/.codex/memories/projects/<customer>/<project>.md`.
- `<root_workspace_path>/.codex/memories/products/<product>.md`.
- `<root_workspace_path>/.codex/memories/patterns/<task_category>.md`.
- `<root_workspace_path>/.codex/memories/active-cases.md`.

Closure summaries must capture symptoms, root cause, fix, validation, reusable commands, risks, related cases, and follow-up actions. Do not store secrets.
