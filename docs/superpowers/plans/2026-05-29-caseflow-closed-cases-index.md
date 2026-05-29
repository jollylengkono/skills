# Caseflow Closed Cases Index Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Split Caseflow closed-case tracking into a permanent `closed-cases.md` central memory index and keep `active-cases.md` free of links to already closed cases.

**Architecture:** This is a documentation-skill behavior update. Source templates define the files Caseflow creates, `caseflow/SKILL.md` defines the workflow rules, and the installed active copy under `/home/jolly_lengkono/.agents/skills/caseflow/` must match the source copy after implementation.

**Tech Stack:** Markdown skill files, shell validation with `rg`, Git commits.

---

## File Structure

- Modify `caseflow/templates/active-cases.md`: remove closed-case sections and make it an active-only operational index.
- Create `caseflow/templates/closed-cases.md`: permanent complete closed-case index template.
- Modify `caseflow/templates/case.md`: update closure checklist items.
- Modify `caseflow/templates/workspace-index.md`: record the split between active and closed indexes.
- Modify `caseflow/SKILL.md`: update directory tree, central memory layout, closure workflow, reopening workflow, follow-up workflow, quality gate, and common mistakes.
- Sync `/home/jolly_lengkono/.agents/skills/caseflow/`: installed active copy must match source after implementation.

## Task 1: Update Source Templates

**Files:**
- Modify: `caseflow/templates/active-cases.md`
- Create: `caseflow/templates/closed-cases.md`
- Modify: `caseflow/templates/case.md`
- Modify: `caseflow/templates/workspace-index.md`

- [ ] **Step 1: Run template checks to capture current failures**

Run:

```bash
rg -n 'Recently Closed|recently closed|Entry moved from `## Active` to `## Recently Closed`|Track active, blocked, recently closed' caseflow/templates
test -f caseflow/templates/closed-cases.md
```

Expected:

```text
caseflow/templates/active-cases.md:7:## Recently Closed
caseflow/templates/active-cases.md:17:- Keep recently closed cases here for the last 10 entries or last 30 days, whichever is more useful for this workspace.
caseflow/templates/case.md:105:- [ ] Entry moved from `## Active` to `## Recently Closed`.
caseflow/templates/workspace-index.md:35:- Track active, blocked, recently closed, reopened, and follow-up-created cases in `active-cases.md`.
```

The `test -f` command should exit non-zero because `caseflow/templates/closed-cases.md` does not exist yet.

- [ ] **Step 2: Replace `caseflow/templates/active-cases.md` with active-only content**

Write this complete file:

```markdown
# Active Cases

## Active

- None recorded yet.

## Blocked

- None recorded yet.

## Reopened

- None recorded yet.

## Operational Rules

- Track only cases whose current operational state is `Active`, `Blocked`, or `Reopened`.
- Do not link to cases that are already closed.
- When a case closes, remove its entry from this file and add or update its permanent entry in `closed-cases.md`.
- When an active follow-up references a closed predecessor, store that relationship in the active case file and `closed-cases.md`, not in this file.

## Markdown Links

- Link to case files using relative Markdown links from this file.
- Include the `.md` filename in every Markdown link.
- Do not store absolute workstation paths for related Markdown files.
```

- [ ] **Step 3: Create `caseflow/templates/closed-cases.md`**

Write this complete file:

````markdown
# Closed Cases

## Closed

- None recorded yet.

## Follow-up Created

- None recorded yet.

## Reopened Notes

- None recorded yet.

## Retention Rules

- Keep this as the permanent complete central index of closed cases.
- Do not prune closed-case history from this file during routine cleanup.
- Do not delete or move case directories when updating this index.
- If a closed case is reopened, keep the original closure entry here and add a note under `## Reopened Notes`.

## Entry Format

Use concise entries with these fields when known:

```markdown
- [Case](../../<customer>/<project>/<case_directory>/<case_directory>.md) | Customer: <customer> | Project: <project> | Product: <product> | Category: <task_category> | Closed: <yyyy-mm-dd> | Status: Closed | Outcome: <root cause or result> | Validation: <validation summary> | Follow-up: <relative link or none>
```

## Markdown Links

- Link to case files using relative Markdown links from this file.
- Include the `.md` filename in every Markdown link.
- Do not store absolute workstation paths for related Markdown files.
````

- [ ] **Step 4: Update `caseflow/templates/case.md` closure checklist**

Replace this checklist item:

```markdown
- [ ] Entry moved from `## Active` to `## Recently Closed`.
```

with these two checklist items:

```markdown
- [ ] Entry removed from `active-cases.md`.
- [ ] Permanent closure entry added or updated in `closed-cases.md`.
```

- [ ] **Step 5: Update `caseflow/templates/workspace-index.md` lifecycle rules**

Replace this rule:

```markdown
- Track active, blocked, recently closed, reopened, and follow-up-created cases in `active-cases.md`.
```

with these two rules:

```markdown
- Track active, blocked, and reopened cases in `active-cases.md`.
- Track closed and follow-up-created closure records permanently in `closed-cases.md`.
```

- [ ] **Step 6: Run source template checks**

Run:

```bash
rg -n 'Recently Closed|recently closed|Entry moved from `## Active` to `## Recently Closed`|Track active, blocked, recently closed' caseflow/templates
test -f caseflow/templates/closed-cases.md
rg -n 'closed-cases.md|Do not link to cases that are already closed|permanent complete central index' caseflow/templates
```

Expected:

```text
The first rg command exits with no matches.
The test command exits 0.
The final rg command shows matches in active-cases.md, closed-cases.md, case.md, and workspace-index.md.
```

- [ ] **Step 7: Commit source template changes**

Run:

```bash
git add caseflow/templates/active-cases.md caseflow/templates/closed-cases.md caseflow/templates/case.md caseflow/templates/workspace-index.md
git commit -m "Update caseflow case index templates"
```

Expected:

```text
Commit succeeds with four template files changed.
```

## Task 2: Update Source Skill Workflow

**Files:**
- Modify: `caseflow/SKILL.md`

- [ ] **Step 1: Run source skill checks to capture current failures**

Run:

```bash
rg -n 'Recently Closed|recently closed|Move the case entry from `## Active` to `## Recently Closed`|active-cases.md' caseflow/SKILL.md
rg -n 'closed-cases.md' caseflow/SKILL.md
```

Expected:

```text
The first rg command shows current active-cases.md and Recently Closed references.
The second rg command exits with no matches.
```

- [ ] **Step 2: Add `closed-cases.md` to the template directory tree**

In the `## Directory Structure` code block, change:

```text
    ├── case.md
```

to:

```text
    ├── case.md
    ├── closed-cases.md
```

- [ ] **Step 3: Add `closed-cases.md` to the central memory layout**

In the `## Central Memory` layout, change:

```text
  active-cases.md
  customers/<customer>.md
```

to:

```text
  active-cases.md
  closed-cases.md
  customers/<customer>.md
```

- [ ] **Step 4: Update new-case central memory seeding**

In the list after "When a case starts", keep the active case update and add the closed case seeding rule immediately after it:

```markdown
- `active-cases.md` with the new case status and relative Markdown link to the case file.
- `closed-cases.md` from template when missing; do not add new active cases to this file.
```

- [ ] **Step 5: Add active-index link restriction to Relative Markdown Links**

After the existing Markdown link examples, add:

```markdown
`active-cases.md` must not link to cases whose current status is `Closed` or `Follow-up Created`. Closed-case links belong in `closed-cases.md`, customer memory, project memory, product memory, pattern memory, or case-local related-case sections.
```

- [ ] **Step 6: Update Key Starting Points closure bullet**

Replace:

```markdown
- Closure: use `Closed Case Lifecycle` and update both local case memory and central memory.
```

with:

```markdown
- Closure: use `Closed Case Lifecycle`, remove closed entries from `active-cases.md`, and update `closed-cases.md` plus reusable central memory.
```

- [ ] **Step 7: Add common mistake for closed links in active index**

Add this bullet under `## Common Mistakes`:

```markdown
- Keeping closed-case links in `active-cases.md` instead of moving closure history to `closed-cases.md`.
```

- [ ] **Step 8: Update closure target files**

In `## Closure`, replace the final central memory bullet:

```markdown
- `<root_workspace_path>/Caseflow/memories/active-cases.md`.
```

with:

```markdown
- `<root_workspace_path>/Caseflow/memories/active-cases.md`.
- `<root_workspace_path>/Caseflow/memories/closed-cases.md`.
```

- [ ] **Step 9: Replace `### Closing a Case` workflow**

Replace the entire `### Closing a Case` numbered list with:

```markdown
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
```

- [ ] **Step 10: Replace `### Recently Closed Retention` with closed index policy**

Replace the entire `### Recently Closed Retention` section with:

```markdown
### Closed Case Index

Keep closed cases in `closed-cases.md` permanently. This file is the complete central index for closed cases and follow-up-created closure records.

When updating closed-case history, add or update the entry in `closed-cases.md`. Do not keep already closed cases in `active-cases.md`. Do not delete or move the case directory or case Markdown file during routine closure indexing.
```

- [ ] **Step 11: Update reopening workflow**

Replace the `### Reopening a Case` numbered list with:

```markdown
Reopen a case only when the same issue, customer, project, product, and work scope resumes. When reopening:

1. Preserve the existing closure summary.
2. Set status to `Reopened`.
3. Add the case back under `## Active` in `active-cases.md`.
4. Add a `Reopen History` entry with date, reason, symptoms, and who requested reopening.
5. Add or update a note in `closed-cases.md` showing that the case was reopened and when.

Keep the original closure entry in `closed-cases.md`. A reopened case may appear in both indexes only while it is reopened: `active-cases.md` tracks the current reopened work, and `closed-cases.md` preserves closure history.
```

- [ ] **Step 12: Update follow-up workflow**

Replace the `When creating a follow-up:` numbered list with:

```markdown
When creating a follow-up:

1. Create a new case using normal Caseflow intake.
2. Set the original case status to `Follow-up Created` if no more work remains in that case.
3. Keep the original closed or follow-up-created case out of `active-cases.md`.
4. Add or update the original case in `closed-cases.md` under `## Follow-up Created`.
5. Link old and new case files both ways with relative Markdown links.
6. Add a short reason in both `Related Cases` sections.
7. Add the new case to `active-cases.md` only if the new follow-up case is active, blocked, or reopened.
8. Update customer, project, product, pattern, and closed-case memory.
```

- [ ] **Step 13: Update closure quality gate**

Replace:

```markdown
- Active case index moved or updated the entry correctly.
```

with:

```markdown
- Active case index no longer links to the closed case.
- Closed case index has a permanent closure entry.
```

- [ ] **Step 14: Run source skill checks**

Run:

```bash
rg -n 'Recently Closed|recently closed|Move the case entry from `## Active` to `## Recently Closed`' caseflow/SKILL.md
rg -n 'closed-cases.md|Closed Case Index|must not link to cases whose current status is `Closed`|Active case index no longer links' caseflow/SKILL.md
```

Expected:

```text
The first rg command exits with no matches.
The second rg command shows the new closed-cases.md references in directory structure, central memory, closure workflow, reopening workflow, follow-up workflow, and quality gate.
```

- [ ] **Step 15: Commit source skill update**

Run:

```bash
git add caseflow/SKILL.md
git commit -m "Update caseflow closed case lifecycle"
```

Expected:

```text
Commit succeeds with caseflow/SKILL.md changed.
```

## Task 3: Sync Installed Skill Copy and Verify

**Files:**
- Modify: `/home/jolly_lengkono/.agents/skills/caseflow/SKILL.md`
- Modify: `/home/jolly_lengkono/.agents/skills/caseflow/templates/active-cases.md`
- Create: `/home/jolly_lengkono/.agents/skills/caseflow/templates/closed-cases.md`
- Modify: `/home/jolly_lengkono/.agents/skills/caseflow/templates/case.md`
- Modify: `/home/jolly_lengkono/.agents/skills/caseflow/templates/workspace-index.md`

- [ ] **Step 1: Sync source files into installed skill copy**

Run:

```bash
cp caseflow/SKILL.md /home/jolly_lengkono/.agents/skills/caseflow/SKILL.md
cp caseflow/templates/active-cases.md /home/jolly_lengkono/.agents/skills/caseflow/templates/active-cases.md
cp caseflow/templates/closed-cases.md /home/jolly_lengkono/.agents/skills/caseflow/templates/closed-cases.md
cp caseflow/templates/case.md /home/jolly_lengkono/.agents/skills/caseflow/templates/case.md
cp caseflow/templates/workspace-index.md /home/jolly_lengkono/.agents/skills/caseflow/templates/workspace-index.md
```

Expected:

```text
All copy commands exit 0.
```

- [ ] **Step 2: Verify source and installed copies match**

Run:

```bash
diff -u caseflow/SKILL.md /home/jolly_lengkono/.agents/skills/caseflow/SKILL.md
diff -u caseflow/templates/active-cases.md /home/jolly_lengkono/.agents/skills/caseflow/templates/active-cases.md
diff -u caseflow/templates/closed-cases.md /home/jolly_lengkono/.agents/skills/caseflow/templates/closed-cases.md
diff -u caseflow/templates/case.md /home/jolly_lengkono/.agents/skills/caseflow/templates/case.md
diff -u caseflow/templates/workspace-index.md /home/jolly_lengkono/.agents/skills/caseflow/templates/workspace-index.md
```

Expected:

```text
All diff commands exit 0 and print no output.
```

- [ ] **Step 3: Run full Caseflow validation checks**

Run:

```bash
rg -n 'Recently Closed|recently closed|Entry moved from `## Active` to `## Recently Closed`|Move the case entry from `## Active` to `## Recently Closed`|Track active, blocked, recently closed' caseflow /home/jolly_lengkono/.agents/skills/caseflow
test -f caseflow/templates/closed-cases.md
test -f /home/jolly_lengkono/.agents/skills/caseflow/templates/closed-cases.md
rg -n 'closed-cases.md|Do not link to cases that are already closed|must not link to cases whose current status is `Closed`|permanent complete central index|Active case index no longer links' caseflow /home/jolly_lengkono/.agents/skills/caseflow
```

Expected:

```text
The first rg command exits with no matches.
Both test commands exit 0.
The final rg command shows matching references in both source and installed skill copies.
```

- [ ] **Step 4: Check git status**

Run:

```bash
git status --short
```

Expected:

```text
The working tree shows no tracked source changes left uncommitted from Tasks 1 and 2.
Untracked docs may still appear if they existed before this implementation.
The installed copy under /home/jolly_lengkono/.agents is outside this repo and will not appear in this git status.
```

- [ ] **Step 5: Report implementation result**

Report:

```text
Updated Caseflow so closed cases are removed from active-cases.md and indexed permanently in closed-cases.md.
Source and installed skill copies were verified with diff and rg checks.
```
