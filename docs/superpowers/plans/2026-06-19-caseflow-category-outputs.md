# Caseflow Category Output Artifacts Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add two category-routed output artifacts to the caseflow skill — a Findings Report (troubleshooting/performance-tuning) and a Task & Blocker Log (installation-configuration/patching/upgrade) — plus the reference workflow, SKILL.md wiring, and a README feature entry.

**Architecture:** Pure documentation/skill-content change to the `caseflow` skill, mirroring the existing Decision Brief feature. Two fill-in templates, one combined reference holding shared rules + per-artifact workflows, a concise SKILL.md section with tree and Key-Starting-Points updates, and a new README feature entry. No code; verification is structural (`grep`).

**Tech Stack:** Markdown only. Repo of record: `/home/jolly_lengkono/skills`, branch `feat/caseflow-category-outputs`.

## Global Constraints

- Artifacts are saved to the case `Output/` folder.
- Findings Report filename: `<case_directory>_findings_<yyyymmdd>.md` (new dated file each generation).
- Task & Blocker Log filename: `<case_directory>_tasks-blockers.md` (one living file per case, updated in place).
- Routing: `troubleshooting`, `performance-tuning` → Findings Report; `installation-configuration`, `patching`, `upgrade` → Task & Blocker Log.
- Sanitization is secrets-only: strip credentials, passwords, wallets, private keys, certificates, production connection strings; keep customer name, hostnames, versions.
- README diagrams are pure ASCII (only `+ - | v < >`, letters, digits, spaces) inside fenced ```text blocks.
- No secrets in any file.

Spec: `docs/superpowers/specs/2026-06-19-caseflow-category-outputs-design.md`.

---

## File Structure

- Create: `caseflow/templates/findings-report.md` — Findings Report fill-in template.
- Create: `caseflow/templates/task-blocker-log.md` — Task & Blocker Log fill-in template.
- Create: `caseflow/references/category-outputs.md` — combined reference: routing table, shared rules (Output/ location, sanitization, linking), and one workflow subsection per artifact.
- Modify: `caseflow/SKILL.md` — Directory Structure tree, new `## Category Output Artifacts` section, Key Starting Points entry.
- Modify: `caseflow/README.md` — new "Category Output Artifacts" feature entry (8th feature) with an ASCII diagram.

**Note on deployment:** The live runtime copy lives at `~/.claude/skills/caseflow/`. Task 6 mirrors the changes there; skip it if your environment syncs automatically.

---

### Task 1: Create the Findings Report template

**Files:**
- Create: `caseflow/templates/findings-report.md`

- [ ] **Step 1: Confirm absent**

Run:
```bash
cd /home/jolly_lengkono/skills
test -f caseflow/templates/findings-report.md && echo EXISTS || echo ABSENT
```
Expected: ABSENT.

- [ ] **Step 2: Create the file with EXACTLY this content**

The angle-bracket fields are intentional fill-in placeholders; keep them.

```markdown
> Sanitized for sharing. No secrets, credentials, wallets, keys, certificates, or production connection strings. Cleared for customer email.

# Findings Report: <case name>

- Case: <case_directory>
- Date: <yyyymmdd>
- Summary: <one-line summary of the issue and outcome>

## Findings

1. <finding>
2. <finding>

## Analysis

1. <analysis for finding 1>
2. <analysis for finding 2>

## Recommendations

1. <recommendation for finding 1>
2. <recommendation for finding 2>
```

- [ ] **Step 3: Structural check**

Run:
```bash
cd /home/jolly_lengkono/skills
grep -c -E '^(# Findings Report:|## Findings|## Analysis|## Recommendations)' caseflow/templates/findings-report.md
```
Expected: `4`.

- [ ] **Step 4: Commit**

```bash
cd /home/jolly_lengkono/skills
git add caseflow/templates/findings-report.md
git commit -m "feat(caseflow): add findings-report template

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 2: Create the Task & Blocker Log template

**Files:**
- Create: `caseflow/templates/task-blocker-log.md`

- [ ] **Step 1: Confirm absent**

Run:
```bash
cd /home/jolly_lengkono/skills
test -f caseflow/templates/task-blocker-log.md && echo EXISTS || echo ABSENT
```
Expected: ABSENT.

- [ ] **Step 2: Create the file with EXACTLY this content**

```markdown
> Sanitized for sharing. No secrets, credentials, wallets, keys, certificates, or production connection strings. Cleared for sharing.

# Task & Blocker Log: <case name>

- Case: <case_directory>
- Last Updated: <yyyymmdd hh:mm>

## Tasks

| # | Task | Status | Start | Done |
|---|------|--------|-------|------|
| 1 | <task> | Pending | | |

Status values: Pending, In progress, Done, Blocked. Start and Done are datetimes (`yyyymmdd hh:mm`).

## Blocker Records

| # | Blocker | Owner | Raised | Needed by | Status | Resolution |
|---|---------|-------|--------|-----------|--------|------------|
| 1 | <blocker> | <owner> | <yyyymmdd> | <yyyymmdd> | Open | |

Blocker status values: Open, Resolved.
```

- [ ] **Step 3: Structural check**

Run:
```bash
cd /home/jolly_lengkono/skills
grep -c -E '^(# Task & Blocker Log:|## Tasks|## Blocker Records)' caseflow/templates/task-blocker-log.md
grep -c -F '| # | Task | Status | Start | Done |' caseflow/templates/task-blocker-log.md
grep -c -F '| # | Blocker | Owner | Raised | Needed by | Status | Resolution |' caseflow/templates/task-blocker-log.md
```
Expected: `3`, then `1`, then `1`.

- [ ] **Step 4: Commit**

```bash
cd /home/jolly_lengkono/skills
git add caseflow/templates/task-blocker-log.md
git commit -m "feat(caseflow): add task-blocker-log template

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 3: Create the combined category-outputs reference

**Files:**
- Create: `caseflow/references/category-outputs.md`

- [ ] **Step 1: Confirm absent**

Run:
```bash
cd /home/jolly_lengkono/skills
test -f caseflow/references/category-outputs.md && echo EXISTS || echo ABSENT
```
Expected: ABSENT.

- [ ] **Step 2: Create the file with EXACTLY this content**

````markdown
# Category Output Artifacts Workflow

Caseflow produces a category-routed output artifact per case, saved to the case
`Output/` folder and linked from the case file. Choose the artifact by the case's
task category.

## Routing

| Task category | Artifact | Section |
|---|---|---|
| `troubleshooting`, `performance-tuning` | Findings Report | Findings Report |
| `installation-configuration`, `patching`, `upgrade` | Task & Blocker Log | Task & Blocker Log |

## Shared Rules

- Save artifacts to the case `Output/` folder.
- Run the Sanitization Gate (below) before saving or sharing.
- Link each artifact from the case file under its Evidence / Outputs area, using a
  relative Markdown link with a short label and the `.md` filename.
- Do not store secrets in any artifact or memory file.

## Sanitization Gate

Reuse caseflow's secrets rule. Strip credentials, passwords, wallets, private
keys, certificates, and production connection strings. Keep customer name,
hostnames, and versions. Keep the top banner stating the artifact is cleared for
sharing. The Findings Report is pasted into customer email, so sanitization is
mandatory before sharing.

## Findings Report

For `troubleshooting` and `performance-tuning` cases. Email-ready, generated on
demand as dated snapshots.

1. Copy `templates/findings-report.md` and fill the header (case name, case
   directory, date, one-line summary).
2. Populate three parallel numbered sections — `## Findings`, `## Analysis`,
   `## Recommendations` — keeping item numbers aligned (Analysis 2 and
   Recommendation 2 address Finding 2).
3. Run the Sanitization Gate.
4. Save to `Output/<case_directory>_findings_<yyyymmdd>.md`. Create a new dated
   file each time so each emailed report is preserved.
5. Link it from the case file with a short label.

## Task & Blocker Log

For `installation-configuration`, `patching`, and `upgrade` cases. A single
living file per case, updated in place.

1. If the log does not exist, copy `templates/task-blocker-log.md` to
   `Output/<case_directory>_tasks-blockers.md`.
2. Maintain the `## Tasks` table (`# | Task | Status | Start | Done`): set Start
   when a task begins and Done when it completes; use status Pending, In
   progress, Done, or Blocked.
3. Maintain the `## Blocker Records` table
   (`# | Blocker | Owner | Raised | Needed by | Status | Resolution`): add a row
   when a blocker arises, set Status to Resolved and fill Resolution when cleared.
4. Run the Sanitization Gate before sharing the file.
5. Link it from the case file with a short label (link once; the file is updated
   in place).
````

- [ ] **Step 3: Structural check**

Run:
```bash
cd /home/jolly_lengkono/skills
grep -c -E '^## (Routing|Shared Rules|Sanitization Gate|Findings Report|Task & Blocker Log)$' caseflow/references/category-outputs.md
```
Expected: `5`.

- [ ] **Step 4: Commit**

```bash
cd /home/jolly_lengkono/skills
git add caseflow/references/category-outputs.md
git commit -m "feat(caseflow): add category-outputs workflow reference

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 4: Wire Category Output Artifacts into SKILL.md

**Files:**
- Modify: `caseflow/SKILL.md` (Directory Structure tree, new section, Key Starting Points)

- [ ] **Step 1: Confirm the section is absent**

Run:
```bash
cd /home/jolly_lengkono/skills
grep -c -E '^## Category Output Artifacts$' caseflow/SKILL.md
```
Expected: `0`.

- [ ] **Step 2: Update the Directory Structure tree**

In `caseflow/SKILL.md`, find this block (it currently has the decision-brief entries):

```text
caseflow/
├── SKILL.md
├── references/
│   └── decision-brief.md
└── templates/
    ├── active-cases.md
    ├── case.md
    ├── closed-cases.md
    ├── customer.md
    ├── decision-brief.md
    ├── pattern.md
    ├── product.md
    ├── project.md
    └── workspace-index.md
```

Replace it with:

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

- [ ] **Step 3: Add the Category Output Artifacts section**

In `caseflow/SKILL.md`, find the line that begins the Decision Brief section:

```text
## Decision Brief
```

Insert the following new section immediately BEFORE that `## Decision Brief` line, leaving a blank line between the new section and `## Decision Brief`:

```markdown
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
```

- [ ] **Step 4: Add a Key Starting Points entry**

In `caseflow/SKILL.md`, find this line in the `## Key Starting Points` list:

```text
- Decision making: use `Decision Brief` to generate a NotebookLM-optimized brief in the case `Output/` folder and link it from the case file `## Decisions` section.
```

Insert this new bullet immediately AFTER it:

```text
- Category output: use `Category Output Artifacts` to produce a Findings Report (troubleshooting/performance-tuning) or Task & Blocker Log (installation-configuration/patching/upgrade) in the case `Output/` folder.
```

- [ ] **Step 5: Structural checks**

Run:
```bash
cd /home/jolly_lengkono/skills
grep -c -E '^## Category Output Artifacts$' caseflow/SKILL.md
grep -c 'references/category-outputs.md' caseflow/SKILL.md
grep -c 'templates/findings-report.md' caseflow/SKILL.md
grep -c 'templates/task-blocker-log.md' caseflow/SKILL.md
grep -c 'Category output: use `Category Output Artifacts`' caseflow/SKILL.md
```
Expected: `1`, `1`, `1`, `1`, `1`.

- [ ] **Step 6: Commit**

```bash
cd /home/jolly_lengkono/skills
git add caseflow/SKILL.md
git commit -m "feat(caseflow): wire category output artifacts into SKILL.md

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 5: Add the README feature entry

**Files:**
- Modify: `caseflow/README.md`

- [ ] **Step 1: Confirm the feature is absent**

Run:
```bash
cd /home/jolly_lengkono/skills
grep -c -E '^### 8\. ' caseflow/README.md
```
Expected: `0`.

- [ ] **Step 2: Add the new feature entry**

In `caseflow/README.md`, find the start of the Directory Structure section:

```text
## Directory Structure
```

Insert the following new feature entry immediately BEFORE that `## Directory Structure` line, leaving a blank line between the new entry and `## Directory Structure`:

```markdown
### 8. Category Output Artifacts

**How it works:** Each case produces a deliverable chosen by its task category.
Troubleshooting and performance-tuning cases get an email-ready Findings Report
(numbered Findings, Analysis, Recommendations); installation, patching, and
upgrade cases get a living Task & Blocker Log. Both save to the case `Output/`
folder and are sanitized before sharing.

**How it helps:** You hand the customer a clean, consistent deliverable straight
from the case — a report you can paste into email, or a live task/blocker tracker
— without re-formatting case notes by hand.

```text
  Task category
     |
     +-- troubleshooting / performance-tuning --> Findings Report
     |        (Findings / Analysis / Recommendations, email-ready)
     |
     +-- installation / patching / upgrade -----> Task & Blocker Log
              (Tasks table + Blocker Records, living file)
                          |
                          v
                 Output/ + linked from case file
```
```

- [ ] **Step 3: Structural checks**

Run:
```bash
cd /home/jolly_lengkono/skills
grep -c -E '^### 8\. Category Output Artifacts$' caseflow/README.md
grep -c -E '^\*\*How it works:\*\*' caseflow/README.md
grep -c -E '^\*\*How it helps:\*\*' caseflow/README.md
grep -c '^```' caseflow/README.md
grep -nP '[\x{2500}-\x{257F}\x{25B6}\x{25BC}\x{2192}\x{2190}]' caseflow/README.md || echo "no-unicode-ok"
```
Expected: `1`, then `8`, then `8`, then `20` (was 18, +2 for the new fenced block), then `no-unicode-ok`.

- [ ] **Step 4: Commit**

```bash
cd /home/jolly_lengkono/skills
git add caseflow/README.md
git commit -m "docs(caseflow): add Category Output Artifacts feature to README

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 6: Mirror changes to the live skills copy

Skip this task if your environment syncs `~/.claude/skills/` automatically.

**Files:**
- Modify: `~/.claude/skills/caseflow/` (live runtime copy)

- [ ] **Step 1: Copy the new and modified files into the live copy**

Run:
```bash
mkdir -p /home/jolly_lengkono/.claude/skills/caseflow/references
cp /home/jolly_lengkono/skills/caseflow/SKILL.md /home/jolly_lengkono/.claude/skills/caseflow/SKILL.md
cp /home/jolly_lengkono/skills/caseflow/README.md /home/jolly_lengkono/.claude/skills/caseflow/README.md
cp /home/jolly_lengkono/skills/caseflow/templates/findings-report.md /home/jolly_lengkono/.claude/skills/caseflow/templates/findings-report.md
cp /home/jolly_lengkono/skills/caseflow/templates/task-blocker-log.md /home/jolly_lengkono/.claude/skills/caseflow/templates/task-blocker-log.md
cp /home/jolly_lengkono/skills/caseflow/references/category-outputs.md /home/jolly_lengkono/.claude/skills/caseflow/references/category-outputs.md
```
Expected: no output.

- [ ] **Step 2: Verify the two caseflow trees match for the touched files**

Run:
```bash
for f in SKILL.md README.md templates/findings-report.md templates/task-blocker-log.md references/category-outputs.md; do
  diff -q "/home/jolly_lengkono/skills/caseflow/$f" "/home/jolly_lengkono/.claude/skills/caseflow/$f"
done && echo "LIVE COPY IN SYNC"
```
Expected: `LIVE COPY IN SYNC` (no diff output). No commit — the live copy is outside the git repo.

---

## Final Verification

- [ ] All three new files exist and pass their structural checks (Tasks 1-3).
- [ ] `caseflow/SKILL.md` has the `## Category Output Artifacts` section, tree entries, and Key Starting Points bullet (Task 4).
- [ ] `caseflow/README.md` has the 8th feature with a pure-ASCII diagram and balanced fences (Task 5).
- [ ] `git log --oneline feat/caseflow-category-outputs ^main` shows the spec commit plus five feature commits.
- [ ] No secrets appear in any new file.
