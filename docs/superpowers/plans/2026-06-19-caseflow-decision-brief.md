# Caseflow Decision Brief Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an on-demand caseflow sub-workflow that generates a single, sanitized, NotebookLM-optimized Markdown "decision brief" per decision, stored in the case `Output/` folder and linked from the case file.

**Architecture:** Pure documentation/skill-content change to the `caseflow` skill. A new template defines the brief's structure (whose heading hierarchy becomes the NotebookLM mind map), a new reference file holds the detailed on-demand workflow, and `caseflow/SKILL.md` gets a concise section plus tree and Key-Starting-Points updates. No code, no API calls; the NotebookLM handoff is a manual file upload by the engineer.

**Tech Stack:** Markdown only. Verification is structural (`grep`/`test` shell checks), not a unit-test framework. Repo of record: `/home/jolly_lengkono/skills`.

---

## File Structure

- Create: `caseflow/templates/decision-brief.md` — the fill-in template for a single decision brief. One responsibility: define the brief's section structure.
- Create: `caseflow/references/decision-brief.md` — the detailed on-demand workflow, NotebookLM optimization notes, sanitization gate. One responsibility: tell the engineer how to produce a brief.
- Modify: `caseflow/SKILL.md` — add the two files to the Directory Structure tree, add a concise "Decision Brief" section, add a Key Starting Points entry. The SKILL.md stays the lean entry point; detail lives in the reference file.

The spec for this plan is `docs/superpowers/specs/2026-06-19-caseflow-decision-brief-design.md`.

**Note on deployment:** The live copy used at runtime lives at `~/.claude/skills/caseflow/` and is currently byte-identical to the repo copy. This plan edits the repo of record. Task 4 mirrors the changes to the live copy so the skill is actually usable; skip Task 4 if your environment syncs automatically.

---

### Task 1: Create the decision-brief template

**Files:**
- Create: `caseflow/templates/decision-brief.md`

- [ ] **Step 1: Write the structural check (expect it to fail — file absent)**

Run:
```bash
cd /home/jolly_lengkono/skills
test -f caseflow/templates/decision-brief.md && \
  grep -c -E '^(# Decision:|## Context|## Options & Trade-offs|## Diagnostic Decision Tree|## Risk & Rollback Gates|## Recommendation & Rationale|## How to use in NotebookLM)' caseflow/templates/decision-brief.md
```
Expected: prints nothing / non-zero exit (file does not exist yet).

- [ ] **Step 2: Create the template file**

Create `caseflow/templates/decision-brief.md` with exactly this content:

```markdown
> Sanitized for external upload. No secrets, credentials, wallets, keys, certificates, or production connection strings. Cleared for NotebookLM.

# Decision: <decision question>

## Context

- Problem or Objective:
- Environment:
- Current State:
- Constraints:

## Options & Trade-offs

### Option 1: <name>

- Summary:
- Pros:
- Cons:
- Effort:
- Risk:

### Option 2: <name>

- Summary:
- Pros:
- Cons:
- Effort:
- Risk:

## Diagnostic Decision Tree

- Symptom: <observed symptom>
  - Hypothesis: <likely cause>
    - Test: <how to confirm>
      - If confirmed: <branch or action>
      - If not confirmed: <next hypothesis>

## Risk & Rollback Gates

- Go / No-Go Conditions:
- Blast Radius:
- Rollback Triggers:
- Rollback Steps:
- Maintenance Window Constraints:

## Recommendation & Rationale

- Recommended Path:
- Rationale:
- Supporting Evidence:
- Rejected Alternatives:

## How to use in NotebookLM

1. Open NotebookLM and create or open a notebook for this case.
2. Upload this Markdown file as a source.
3. Open the Mind Map view to explore the decision branches.
```

- [ ] **Step 3: Run the structural check (expect it to pass)**

Run:
```bash
cd /home/jolly_lengkono/skills
grep -c -E '^(# Decision:|## Context|## Options & Trade-offs|## Diagnostic Decision Tree|## Risk & Rollback Gates|## Recommendation & Rationale|## How to use in NotebookLM)' caseflow/templates/decision-brief.md
```
Expected: prints `7` (all seven required headings present).

- [ ] **Step 4: Commit**

```bash
cd /home/jolly_lengkono/skills
git add caseflow/templates/decision-brief.md
git commit -m "feat(caseflow): add decision-brief template

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 2: Create the decision-brief workflow reference

**Files:**
- Create: `caseflow/references/decision-brief.md`

- [ ] **Step 1: Write the structural check (expect it to fail — file absent)**

Run:
```bash
cd /home/jolly_lengkono/skills
test -f caseflow/references/decision-brief.md && \
  grep -c -E '^## (When to Use|Workflow|Sanitization Gate|NotebookLM Optimization|Storage and Linking)' caseflow/references/decision-brief.md
```
Expected: prints nothing / non-zero exit (file does not exist yet).

- [ ] **Step 2: Create the reference file**

Create `caseflow/references/decision-brief.md` with exactly this content:

````markdown
# Decision Brief Workflow

A decision brief is a single, focused, NotebookLM-optimized Markdown document for
one decision in an active case. Caseflow owns the source document; NotebookLM
owns the mind map. The handoff is a manual file upload by the engineer. There is
no API call to NotebookLM.

## When to Use

On-demand, whenever a real decision arises during an active case (for example:
patch vs upgrade, which fix to apply, go/no-go before a change window). Generate
one brief per decision. Regenerate or add new briefs as the case evolves.

## Workflow

1. Confirm the decision question and a short `<slug>` (1-3 words, kebab-case,
   e.g. `patch-vs-upgrade`).
2. Read the existing case file `<case_directory>.md` and pull relevant Context,
   Findings, and Decisions.
3. Copy `caseflow/templates/decision-brief.md` and populate every branch:
   Context, Options & Trade-offs, Diagnostic Decision Tree, Risk & Rollback
   Gates, Recommendation & Rationale.
4. Run the Sanitization Gate (below) before saving.
5. Save the populated file into the case `Output/` folder as:

   ```text
   <case_directory>_decision_<slug>_<yyyymmdd>.md
   ```

   Use the local current date in `yyyymmdd` format. One decision per file.
6. Add a one-line link to the brief under the case file's `## Decisions`
   section, using a relative Markdown link with a short reason. Follow caseflow's
   relative-link rules and include the `.md` filename. Example:

   ```markdown
   - [Patch vs upgrade decision](Output/<case_directory>_decision_patch-vs-upgrade_<yyyymmdd>.md) — chose path for listener fix.
   ```
7. Report the generated path and remind the engineer to upload it to NotebookLM
   (see How to use in NotebookLM in the generated file).

## Sanitization Gate

The brief leaves for an external Google service, so sanitization is mandatory
before saving. Reuse caseflow's existing secrets rule:

- Strip credentials, passwords, wallets, private keys, certificates, and
  production connection strings.
- Keep customer name, hostnames, and versions — these are needed for a useful
  decision map.
- Keep the top banner stating the brief is cleared for external upload.

## NotebookLM Optimization

- The heading hierarchy IS the mind map. Keep the template's `#`/`##`/`###`
  structure so the map branches as intended.
- Title (`# Decision: ...`) is the central node — phrase it as the decision
  question.
- One decision per document keeps the central node and branches clean.
- Provide the short Context block so the map is grounded, but do not paste the
  whole case log — pull only what the decision needs.

## Storage and Linking

- The brief is an output artifact: store it under the case `Output/` folder.
- Link each brief from the case file `## Decisions` section with a short reason.
- Do not store secrets in the brief or in any memory file.
````

- [ ] **Step 3: Run the structural check (expect it to pass)**

Run:
```bash
cd /home/jolly_lengkono/skills
grep -c -E '^## (When to Use|Workflow|Sanitization Gate|NotebookLM Optimization|Storage and Linking)' caseflow/references/decision-brief.md
```
Expected: prints `5` (all five required section headings present).

- [ ] **Step 4: Commit**

```bash
cd /home/jolly_lengkono/skills
git add caseflow/references/decision-brief.md
git commit -m "feat(caseflow): add decision-brief workflow reference

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 3: Wire the Decision Brief into SKILL.md

**Files:**
- Modify: `caseflow/SKILL.md` (Directory Structure tree, new section, Key Starting Points)

- [ ] **Step 1: Write the structural check (expect it to fail — section absent)**

Run:
```bash
cd /home/jolly_lengkono/skills
grep -c -E '^## Decision Brief$' caseflow/SKILL.md
```
Expected: prints `0` (section not added yet).

- [ ] **Step 2: Update the Directory Structure tree**

In `caseflow/SKILL.md`, find this block:

```text
caseflow/
├── SKILL.md
└── templates/
    ├── active-cases.md
    ├── case.md
    ├── closed-cases.md
    ├── customer.md
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

- [ ] **Step 3: Add the Decision Brief section**

In `caseflow/SKILL.md`, find the start of this section:

```text
## Key Starting Points
```

Insert the following new section immediately BEFORE that `## Key Starting Points` line (leave a blank line between the new section and `## Key Starting Points`):

```markdown
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
3. Populate `templates/decision-brief.md` (Context, Options & Trade-offs,
   Diagnostic Decision Tree, Risk & Rollback Gates, Recommendation & Rationale).
4. Sanitize: strip secrets, credentials, wallets, keys, certificates, and
   production connection strings; keep customer name, hostnames, and versions.
5. Save to the case `Output/` folder as
   `<case_directory>_decision_<slug>_<yyyymmdd>.md`. One decision per file.
6. Link the brief from the case file `## Decisions` section with a relative
   Markdown link and a short reason.

See `references/decision-brief.md` for the full workflow and NotebookLM
optimization notes.
```

- [ ] **Step 4: Add a Key Starting Points entry**

In `caseflow/SKILL.md`, find this line in the `## Key Starting Points` list:

```text
- Closure: use `Closed Case Lifecycle`, remove closed entries from `active-cases.md`, and update `closed-cases.md` plus reusable central memory.
```

Insert this new bullet immediately AFTER it:

```text
- Decision making: use `Decision Brief` to generate a NotebookLM-optimized brief in the case `Output/` folder and link it from the case file `## Decisions` section.
```

- [ ] **Step 5: Run the structural checks (expect them to pass)**

Run:
```bash
cd /home/jolly_lengkono/skills
grep -c -E '^## Decision Brief$' caseflow/SKILL.md
grep -c 'references/decision-brief.md' caseflow/SKILL.md
grep -c 'templates/decision-brief.md' caseflow/SKILL.md
grep -c 'Decision making: use `Decision Brief`' caseflow/SKILL.md
```
Expected: prints `1`, then `1`, then `1`, then `1` (one per command).

- [ ] **Step 6: Commit**

```bash
cd /home/jolly_lengkono/skills
git add caseflow/SKILL.md
git commit -m "feat(caseflow): wire decision brief workflow into SKILL.md

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 4: Mirror changes to the live skills copy

Skip this task if your environment syncs `~/.claude/skills/` from the repo automatically.

**Files:**
- Modify: `~/.claude/skills/caseflow/` (live runtime copy)

- [ ] **Step 1: Verify the live copy is currently in sync (sanity check before overwriting)**

Run:
```bash
diff -r /home/jolly_lengkono/skills/caseflow /home/jolly_lengkono/.claude/skills/caseflow
```
Expected: differences ONLY for the three new/changed paths (`SKILL.md`, `templates/decision-brief.md`, `references/decision-brief.md`). If you see unrelated differences, stop and report them — do not overwrite.

- [ ] **Step 2: Copy the new and modified files into the live copy**

Run:
```bash
mkdir -p /home/jolly_lengkono/.claude/skills/caseflow/references
cp /home/jolly_lengkono/skills/caseflow/SKILL.md /home/jolly_lengkono/.claude/skills/caseflow/SKILL.md
cp /home/jolly_lengkono/skills/caseflow/templates/decision-brief.md /home/jolly_lengkono/.claude/skills/caseflow/templates/decision-brief.md
cp /home/jolly_lengkono/skills/caseflow/references/decision-brief.md /home/jolly_lengkono/.claude/skills/caseflow/references/decision-brief.md
```
Expected: no output (success).

- [ ] **Step 3: Verify the live copy now matches the repo**

Run:
```bash
diff -r /home/jolly_lengkono/skills/caseflow /home/jolly_lengkono/.claude/skills/caseflow
```
Expected: no output (live copy is byte-identical to the repo). No commit — the live copy is outside the git repo.

---

## Final Verification

- [ ] All three repo files exist and pass their structural checks (re-run the grep checks from Tasks 1-3).
- [ ] `git log --oneline -4` shows the three feature commits.
- [ ] The case template's `## Decisions` section is unchanged (we link from it, we do not modify it).
- [ ] No secrets appear in any new file.
