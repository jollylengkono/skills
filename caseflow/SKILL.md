---
name: caseflow
description: Use when starting, organizing, continuing, or closing Oracle customer work cases that need persistent case memory, customer/project folder structure, product skill routing, related-case lookup, or closure summarization.
---

# Caseflow

Caseflow is the intake and memory workflow for Oracle Advanced Services engagements. It creates a consistent customer/project/case workspace, maintains local and central memory, routes to existing Oracle product skills, and links related cases.

## How to Use This Skill

1. Use Caseflow before creating or continuing an Oracle customer work case.
2. Complete the required intake fields before creating folders or memory files.
3. Create case-local memory from `templates/case.md`, saved as `<case_directory_name>.md` inside the case directory.
4. Create or update central memory under `<root_workspace_path>/Caseflow/memories/`.
5. Route to the product skill listed in `Product Routing`; if no product skill exists, follow the missing-skill fallback.
6. Update local and central memory when the case changes or closes.

## Directory Structure

```text
caseflow/
├── SKILL.md
├── references/
│   ├── category-outputs.md
│   └── decision-brief.md
└── templates/
    ├── active-cases.md
    ├── case.md
    ├── closed-cases.md
    ├── customer.md
    ├── decision-brief.md
    ├── findings-report.md
    ├── pattern.md
    ├── product.md
    ├── project.md
    ├── task-blocker-log.md
    └── workspace-index.md
```

## Required Inputs

Ask the engineer for these values before creating files:

1. Full root workspace path, unless Caseflow can discover it by walking upward from the current directory to `Caseflow/memories/workspace-index.md`.
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

1. Resolve the root workspace path first.
2. Before asking for the root path, walk upward from the current directory until `Caseflow/memories/workspace-index.md` is found.
3. If found, use the parent directory containing `Caseflow/` as the root workspace path.
4. If not found, ask the engineer for the full root workspace path.
5. Confirm the selected root workspace path exists.
6. Search one level below root for a matching customer directory.
7. If exactly one likely customer match exists, use it.
8. If multiple likely customer matches exist, ask the engineer to choose.
9. If no customer match exists, ask before creating a new customer directory.
10. Repeat the same resolution under the customer directory for the project directory.

## New vs Continuing Case

- New case: resolve root, resolve or create customer and project, collect remaining required inputs, create case structure, and seed or update central memory.
- Continuing case: resolve root by upward discovery, identify the existing case directory, read local `<case_directory_name>.md`, load relevant central memory, and update existing records instead of creating a new case folder or asking for a short case name.
- Legacy case: if `<case_directory_name>.md` is absent but `case.md` exists, read `case.md` and ask before renaming or copying it to the case-named file.

## Case Structure

Create the case folder as:

```text
<root_workspace_path>/<customer>/<project>/<yyyymmdd>_<product_abbrev>_<short_case_name>/
```

Create these entries inside it:

```text
<yyyymmdd>_<product_abbrev>_<short_case_name>.md
Reference/
Logs/
Docs/
Screenshots/
Scripts/
Output/
```

Use the local current date in `yyyymmdd` format. Use underscores in generated case folder names. Lowercase product abbreviation and short case name. Preserve existing customer and project directory spelling.

## Case File Naming

Keep the repository template named `caseflow/templates/case.md`. When creating a real case, copy that template to the generated case directory using the case directory name:

```text
<root_workspace_path>/<customer>/<project>/<case_directory>/<case_directory>.md
```

For example, a case directory named `20260527_db_listener_issue` must contain `20260527_db_listener_issue.md`.

Do not create new cases with a generic `case.md`. For existing cases that already use `case.md`, treat it as legacy format and ask before renaming, copying, or merging.

## Central Memory

Maintain central memory under the resolved root workspace path. Do not write central memory relative to the current shell directory unless it is the selected root workspace.

```text
<root_workspace_path>/Caseflow/memories/
  workspace-index.md
  active-cases.md
  closed-cases.md
  customers/<customer>.md
  projects/<customer>/<project>.md
  products/<product>.md
  patterns/<task_category>.md
```

Persist related Markdown references as relative Markdown links. Keep plain root-relative paths only as secondary human-readable identifiers when useful, for example `Mandiri/New_MCM/20260523_jvm_heap_full_gc`.

Use absolute paths for runtime filesystem operations, but do not make absolute workstation-specific paths the primary link format in reusable central memory.

If `<root_workspace_path>/.caseflow/memories/` exists but `<root_workspace_path>/Caseflow/memories/` does not, do not silently migrate existing memory. Ask whether to create `Caseflow/memories/` for new Caseflow memory, keep reading the legacy `.caseflow/memories/` tree, or perform a separate migration step.

If `<root_workspace_path>/.codex/memories/` exists but `<root_workspace_path>/Caseflow/memories/` does not, do not silently migrate existing memory. Ask whether to create `Caseflow/memories/` for new Caseflow memory or whether the engineer wants a separate migration step before reading or copying old memory.

Case-local memory records what happened. Central memory records reusable knowledge.

When a case starts, create missing central memory directories and seed missing files from templates. Update:

- `workspace-index.md` with the root workspace, customer, project, and product coverage.
- `active-cases.md` with the new case status and relative Markdown link to the case file.
- `closed-cases.md` from template when missing; do not add new active cases to this file.
- `customers/<customer>.md` with the project and relative Markdown link to the case file.
- `projects/<customer>/<project>.md` with product, environment if known, and relative Markdown link to the case file.
- `products/<product>.md` with the relative Markdown link to the case file and reusable product context.
- `patterns/<task_category>.md` with the relative Markdown link to the case file and category-specific evidence needs.

## Relative Markdown Links

All persisted references from memory files or case files to related Markdown files must be relative Markdown links. Use absolute paths only while reading, writing, scanning, or executing filesystem operations.

Rules:

- In `Caseflow/memories/*.md`, link to case files relative to the memory file location.
- In a case file, link back to central memory files relative to the case file location.
- Include the `.md` filename in every Markdown link.
- Do not persist workstation-specific absolute paths such as `/home/<user>/...` in case or memory links.
- If a related artifact is not Markdown, record the relative path as plain text unless a Markdown link is useful.

Examples:

```markdown
[Case](../../<customer>/<project>/<case_directory>/<case_directory>.md)
[Customer Memory](../../../Caseflow/memories/customers/<customer>.md)
```

`active-cases.md` must not link to cases whose current status is `Closed` or `Follow-up Created`. Closed-case links belong in `closed-cases.md`, customer memory, project memory, product memory, pattern memory, or case-local related-case sections.

Before closing a case or standardizing a directory, scan touched Markdown files for absolute links and convert them to relative links.

## Standardize Existing Directories

Use this workflow when the engineer points to an existing directory and asks Caseflow to standardize it.

1. Confirm the directory path exists.
2. Scan the directory tree and classify likely root, customer, project, case, evidence, and memory paths.
3. Report a normalization plan before moving, renaming, or writing files.
4. Preserve existing files and evidence directories. Do not delete or overwrite.
5. Ensure central memory is under `<root_workspace_path>/Caseflow/memories/`.
6. Ensure customer/project/case nesting follows `<root_workspace_path>/<customer>/<project>/<case_directory>/`.
7. Ensure each case directory has `<case_directory>.md`; if only `case.md` exists, ask before copying or renaming.
8. Seed missing central memory files from templates.
9. Convert Markdown references in memory and case files to relative links.
10. Record unresolved conflicts or skipped changes in a migration note inside the relevant case directory or `Caseflow/memories/workspace-index.md`.

If a target path already exists, ask the engineer before merging content. If a safe automatic action is unclear, leave the original file in place and document the required manual decision.

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
4. Record the missing product skill under a `Skill Gaps` section in `<root_workspace_path>/Caseflow/memories/workspace-index.md`.
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

## Category Output Artifacts

Each case produces a category-routed output artifact in the case `Output/` folder,
linked from the case file. Choose the artifact by the case's task category.

| Task category | Artifact |
|---|---|
| `troubleshooting`, `performance-tuning` | Findings Report |
| `installation-configuration`, `patching`, `upgrade` | Task & Blocker Log |

- Findings Report — email-ready, three parallel numbered sections (Findings,
  Analysis, Recommendations) with aligned item numbers. Generate on demand as a
  dated snapshot: `Output/<case_directory>_findings_<yyyymmdd>.md`. Populate from
  `templates/findings-report.md`.
- Task & Blocker Log — a single living file per case updated in place:
  `Output/<case_directory>_tasks-blockers.md`. Tasks table tracks Status, Start,
  and Done; Blocker Records table tracks owner, dates, status, and resolution.
  Populate from `templates/task-blocker-log.md`.

Sanitize before saving or sharing (secrets only). Link the artifact from the case
file with a relative Markdown link. See `references/category-outputs.md` for the
full workflow.

## Decision Brief

When the engineer needs to make a decision during an active case (for example
patch vs upgrade, which fix to apply, or go/no-go before a change window),
generate a decision brief on demand.

A decision brief is a single, sanitized, NotebookLM-optimized Markdown document
for one decision. Caseflow produces the source document; the engineer uploads it
to NotebookLM, which renders the mind map. There is no API call to NotebookLM.

Steps:

1. Confirm the decision question and a short `<slug>` (1-3 words, kebab-case).
2. Pull relevant Context, Findings, and Decisions from the case file.
3. Copy `templates/decision-brief.md` and populate every branch (Context,
   Options & Trade-offs, Diagnostic Decision Tree, Risk & Rollback Gates,
   Recommendation & Rationale).
4. Sanitize: strip secrets, credentials, wallets, keys, certificates, and
   production connection strings; keep customer name, hostnames, and versions.
5. Save to the case `Output/` folder as
   `<case_directory>_decision_<slug>_<yyyymmdd>.md`. One decision per file.
6. Link the brief from the case file `## Decisions` section with a relative
   Markdown link and a short reason.
7. Report the saved file path and remind the engineer to upload it to NotebookLM
   (see the `## How to use in NotebookLM` section in the generated brief).

See `references/decision-brief.md` for the full workflow and NotebookLM
optimization notes.

## Key Starting Points

- New case intake: start with `Required Inputs`, then `Directory Resolution`.
- Product routing: use `Product Routing`, then load only the matched product skill files.
- Memory setup: use `Central Memory` and the templates under `caseflow/templates/`.
- Existing directory cleanup: use `Standardize Existing Directories`.
- Similar prior work: use `Related Cases` and scan customer, project, product, and pattern memory.
- Closure: use `Closed Case Lifecycle`, remove closed entries from `active-cases.md`, and update `closed-cases.md` plus reusable central memory.
- Decision making: use `Decision Brief` to generate a NotebookLM-optimized brief in the case `Output/` folder and link it from the case file `## Decisions` section.
- Category output: use `Category Output Artifacts` to produce a Findings Report (troubleshooting/performance-tuning) or Task & Blocker Log (installation-configuration/patching/upgrade) in the case `Output/` folder.

## Common Mistakes

- Writing central memory under the shell's current directory instead of `<root_workspace_path>/Caseflow/memories/`.
- Creating a case directly under the customer directory and skipping the project layer.
- Creating new cases with generic `case.md` instead of `<case_directory_name>.md`.
- Persisting absolute Markdown links instead of relative Markdown links.
- Routing directly to a nested Fusion skill without first checking `fusion/SKILL.md`.
- Linking every same-product case instead of adding only high-signal related cases.
- Keeping closed-case links in `active-cases.md` instead of moving closure history to `closed-cases.md`.
- Storing secrets, passwords, wallets, certificates, or production connection strings in central memory.

## Closure

When the engineer says the case is closed, update:

- Local `<case_directory_name>.md`.
- `<root_workspace_path>/Caseflow/memories/customers/<customer>.md`.
- `<root_workspace_path>/Caseflow/memories/projects/<customer>/<project>.md`.
- `<root_workspace_path>/Caseflow/memories/products/<product>.md`.
- `<root_workspace_path>/Caseflow/memories/patterns/<task_category>.md`.
- `<root_workspace_path>/Caseflow/memories/active-cases.md`.
- `<root_workspace_path>/Caseflow/memories/closed-cases.md`.

Closure summaries must capture symptoms, root cause, fix, validation, reusable commands, risks, related cases, and follow-up actions. Do not store secrets.

## Closed Case Lifecycle

Keep closed case directories in place. Do not move closed cases to an archive directory unless the engineer explicitly asks for an archive migration and approves the link rewrite plan. Stable case paths are more important than a visually clean workspace.

Allowed case statuses:

| Status | Meaning |
| --- | --- |
| `Active` | Work is ongoing. |
| `Blocked` | Work is paused on missing input, access, approval, or external action. |
| `Closed` | Work is complete, validated, and summarized. |
| `Reopened` | A closed case resumed because the same scope or issue returned. |
| `Follow-up Created` | A new related case was opened because the new work has different scope, timing, product, or risk. |

### Closing a Case

When closing a case:

1. Confirm the engineer explicitly says the case is closed or approves closure.
2. Update local `<case_directory_name>.md`:
   - Set `Status: Closed`.
   - Record `Closed Date`.
   - Complete `Closure Summary`.
   - Complete `Closure Checklist`.
3. Remove the case entry from `Caseflow/memories/active-cases.md`.
4. Add or update a permanent entry in `Caseflow/memories/closed-cases.md`.
5. Add or update closure notes in customer, project, product, and pattern memory.
6. Capture reusable commands, validated fixes, risks, and follow-up actions.
7. Verify touched Markdown links are relative and include `.md` filenames.
8. Do not store secrets, credentials, private keys, wallets, or production connection strings.

### Closed Case Index

Keep closed cases in `closed-cases.md` permanently. This file is the complete central index for closed cases and follow-up-created closure records.

When updating closed-case history, add or update the entry in `closed-cases.md`. Do not keep already closed cases in `active-cases.md`. Do not delete or move the case directory or case Markdown file during routine closure indexing.

### Reopening a Case

Reopen a case only when the same issue, customer, project, product, and work scope resumes. When reopening:

1. Preserve the existing closure summary.
2. Set status to `Reopened`.
3. Add the case back under `## Reopened` in `active-cases.md`.
4. Add a `Reopen History` entry with date, reason, symptoms, and who requested reopening.
5. Add or update a note in `closed-cases.md` showing that the case was reopened and when.

Keep the original closure entry in `closed-cases.md`. A reopened case may appear in both indexes only while it is reopened: `active-cases.md` tracks the current reopened work, and `closed-cases.md` preserves closure history.

If the new work has different scope, different product, a new maintenance window, or separate risk, create a follow-up case instead of reopening.

### Follow-up Cases

Create a follow-up case when new work is related but should be tracked separately. Common reasons:

- New scope or objective.
- Different Oracle product or component.
- New maintenance window or approval chain.
- Different risk profile.
- Post-closure enhancement, hardening, or preventive action.

When creating a follow-up:

1. Create a new case using normal Caseflow intake.
2. Set the original case status to `Follow-up Created` if no more work remains in that case.
3. Keep the original closed or follow-up-created case out of `active-cases.md`.
4. Add or update the original case in `closed-cases.md` under `## Follow-up Created`.
5. Link old and new case files both ways with relative Markdown links.
6. Add a short reason in both `Related Cases` sections.
7. Add the new case to `active-cases.md` only if the new follow-up case is active, blocked, or reopened.
8. Update customer, project, product, pattern, and closed-case memory.

### Blocked Cases

Use `Blocked` when a case is not closed but cannot progress. Record blocker, owner, requested input, requested date, and next review date. Keep blocked cases under `## Blocked` with a visible blocked note.

### Closure Quality Gate

Before reporting that a case is closed, verify:

- Local case file status and closure date are updated.
- Active case index no longer links to the closed case.
- Closed case index has a permanent closure entry.
- Customer, project, product, and pattern memory have reusable closure notes where relevant.
- Related cases are linked selectively with reasons.
- Follow-up actions are either recorded or converted into follow-up cases.
- All touched Markdown references are relative links.
- No secrets or sensitive connection details were added.

## Sources

- `README.md`
- `SKILL_AUTHORING_GUIDE.md`
- `db/SKILL.md`
- `apex/SKILL.md`
- `fusion/SKILL.md`
- `oci/SKILL.md`
- `oem/SKILL.md`
- `graal/SKILL.md`
