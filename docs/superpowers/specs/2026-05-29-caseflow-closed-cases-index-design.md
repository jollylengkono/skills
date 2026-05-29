# Caseflow Closed Cases Index Design

## Context

Caseflow currently treats `active-cases.md` as a mixed operational index. Its template includes active cases, recently closed cases, and follow-up-created cases. The closure workflow says to move closed entries from `## Active` to `## Recently Closed` in `Caseflow/memories/active-cases.md`.

That makes the active index less clear and keeps links to already closed cases in the same central file used to find work that still needs attention.

## Goals

- Add a separate permanent central memory file named `closed-cases.md`.
- Keep `active-cases.md` focused on cases that are operationally active.
- Remove links to already closed cases from `active-cases.md`.
- Preserve stable case directories and case Markdown paths.
- Keep all persisted Markdown references relative and include `.md` filenames.

## Non-Goals

- Move closed case folders to an archive directory.
- Delete closed case directories or case Markdown files.
- Automatically migrate existing workspaces during the skill update.
- Change product routing, task categories, evidence folders, or case-local memory structure beyond closure checklist wording.

## Central Memory Layout

Caseflow central memory should include both active and closed case indexes:

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

`closed-cases.md` is a permanent complete index. It is not a rolling "recently closed" list.

## Active Case Index

`active-cases.md` should only track cases whose current operational state is one of:

- `Active`
- `Blocked`
- `Reopened`

`active-cases.md` must not contain links to cases that are already closed. It should not have a `Recently Closed` section.

If an active follow-up case needs to reference a closed predecessor, that relationship belongs in the active case's local case file and in `closed-cases.md`, not in `active-cases.md`.

## Closed Case Index

Create a new template at:

```text
caseflow/templates/closed-cases.md
```

The template should include:

- `# Closed Cases`
- A `## Closed` section for permanent closed-case entries.
- A `## Follow-up Created` section for cases that are complete but resulted in separately tracked follow-up work.
- A `## Reopened Notes` section to preserve the fact that a closed case was later reopened without removing the original closure entry.
- A `## Retention Rules` section stating that this is a permanent complete index and pruning should not remove closed-case history.
- A `## Markdown Links` section requiring relative Markdown links and prohibiting absolute workstation paths.

Closed entries should be concise. Each entry should include the case link, customer, project, product, task category, closed date, closure status, root cause or outcome, validation summary, and follow-up link if applicable.

## Closure Workflow

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
6. Capture reusable commands, validated fixes, risks, related cases, and follow-up actions.
7. Verify touched Markdown links are relative and include `.md` filenames.
8. Do not store secrets, credentials, private keys, wallets, or production connection strings.

## Reopening Workflow

Reopen a case only when the same issue, customer, project, product, and work scope resumes.

When reopening:

1. Preserve the existing closure summary in the local case file.
2. Set status to `Reopened`.
3. Add the case back under `## Active` in `active-cases.md`.
4. Add a `Reopen History` entry in the local case file.
5. Add or update a note in `closed-cases.md` showing that the case was reopened and when.

The original closure entry should remain in `closed-cases.md`. A reopened case may appear in both indexes only while it is reopened: `active-cases.md` tracks the current reopened work, and `closed-cases.md` preserves closure history.

## Follow-up Cases

Create a follow-up case when new work is related but should be tracked separately because scope, timing, product, approval, or risk differs.

When a follow-up is created from a closed or closing case:

- Keep the original closed case out of `active-cases.md`.
- Add or update the original case in `closed-cases.md` under `## Follow-up Created` if no more work remains in the original case.
- Link old and new case files both ways with relative Markdown links.
- Add the new case to `active-cases.md` only if the new follow-up case is active, blocked, or reopened.

## Template Updates

`active-cases.md` template should remove:

- `## Recently Closed`
- recently closed retention rules
- any instruction that keeps closed entries in the active index

`case.md` closure checklist should replace:

```text
- [ ] Entry moved from `## Active` to `## Recently Closed`.
```

with:

```text
- [ ] Entry removed from `active-cases.md`.
- [ ] Permanent closure entry added or updated in `closed-cases.md`.
```

`workspace-index.md` should state that active, blocked, and reopened cases are tracked in `active-cases.md`, while closed and follow-up-created closure records are tracked permanently in `closed-cases.md`.

`SKILL.md` should update every closure, lifecycle, quality-gate, and central-memory reference so the split is consistent.

## Error Handling

- If `closed-cases.md` is missing when closing a case, create it from the template before updating closure records.
- If a closed entry already exists in `active-cases.md`, remove it during closure or standardization and ensure it exists in `closed-cases.md`.
- If an entry's current status is ambiguous, inspect the local case file before moving it between indexes.
- If moving an entry risks losing information, copy the information into `closed-cases.md` and leave a short cleanup note in `workspace-index.md`.

## Testing

Validate the update with targeted checks:

- Search `caseflow` for remaining `Recently Closed` references.
- Search `caseflow` for instructions that move closed cases into `active-cases.md`.
- Confirm `closed-cases.md` appears in the central memory layout, closure workflow, reopening workflow, quality gate, and templates directory listing.
- Confirm `active-cases.md` does not include closed-case sections or retention rules.
- Confirm the case template checklist references both removing from `active-cases.md` and updating `closed-cases.md`.
- Confirm both source and installed skill copies are updated when implementation proceeds.

## Implementation Scope

The implementation should update:

- `caseflow/SKILL.md`
- `caseflow/templates/active-cases.md`
- `caseflow/templates/case.md`
- `caseflow/templates/workspace-index.md`
- `caseflow/templates/closed-cases.md`

After source updates, refresh the installed active copy under `/home/jolly_lengkono/.agents/skills/caseflow/` so future Caseflow invocations use the new closure behavior.
