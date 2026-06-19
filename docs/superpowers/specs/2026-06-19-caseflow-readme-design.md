# Caseflow README — Design

Date: 2026-06-19
Status: Approved (pending implementation plan)

## Goal

Create `caseflow/README.md`, a human-facing overview of the caseflow skill with
two required parts:

1. A feature list (curated core of 7), each entry giving a short how-it-works, a
   how-it-helps, and a simple ASCII architecture diagram.
2. A directory-structure section documenting what the skill creates on disk, with
   a one-line description per directory.

The README is the human-facing overview. `SKILL.md` remains the authoritative
behavior spec; the README points to it rather than duplicating the rules.

## Scope

In scope:

- Create `caseflow/README.md` only.

Out of scope (YAGNI):

- No changes to `caseflow/SKILL.md`, templates, or references.
- No PNG/Mermaid assets — diagrams are ASCII inside fenced ```text blocks.
- No duplication of SKILL.md's detailed rules (relative-link rules, exact
  closure checklists, etc.). The README summarizes and links to SKILL.md.

## Format Decisions

- **Diagrams:** ASCII art inside fenced ```text blocks. Chosen for maximum
  portability — renders identically on GitHub, in a terminal, and in any editor.
- **Structure:** feature-first reference (intro → features → directory structure
  → how-to-use pointer). Scannable as a lookup reference.

## Document Structure

```text
# Caseflow
<1-paragraph intro: what caseflow is and what it gives you>

## Features
### 1..7  (each uses the per-feature template below)

## Directory Structure
### Case Workspace      (annotated ASCII tree)
### Central Memory      (annotated ASCII tree)

## How to Use
<short pointer: invoke the skill; SKILL.md is the authoritative spec>
```

## Per-Feature Template

Every feature entry uses the same shape for consistency:

```text
### N. <Feature name>

**How it works:** 1-2 sentences.

**How it helps:** 1-2 sentences describing the user benefit.

​```text
<simple ASCII architecture diagram>
​```
```

## The Seven Features

Content for each is summarized from `caseflow/SKILL.md` (the authoritative
source). Each gets a how-it-works line, a how-it-helps line, and an ASCII diagram.

1. **Case Intake & Workspace Scaffolding** — collect required inputs (customer,
   project, task category, Oracle product, short case name); create the dated,
   named case folder with evidence subfolders; seed case-local memory from
   template. Helps: every case lands in the same predictable place.
   Diagram: Engineer → Caseflow → case folder + subfolders.

2. **Directory Resolution** — walk upward from the current directory to find
   `Caseflow/memories/workspace-index.md`; resolve root, then customer, then
   project; ask only when ambiguous. Helps: run caseflow from anywhere in the
   workspace without re-entering the root path.
   Diagram: cwd → walk up → root/customer/project resolved.

3. **Two-Tier Memory** — case-local `<case>.md` records what happened on this
   case; central `Caseflow/memories/` records reusable knowledge across
   customers, projects, products, and patterns. Helps: per-case detail plus
   cross-case reuse without copy-paste.
   Diagram: Case file ↔ Central memory (customers/projects/products/patterns).

4. **Product Skill Routing** — map the Oracle product (and task category) to the
   matching local product skill (db, apex, oci, oem, fusion subskills); fall back
   to the closest domain skill and record skill gaps when none exists. Helps:
   the right product playbook is loaded automatically.
   Diagram: Product + task category → routing table → product skill.

5. **Related Cases Linking** — link cases that share a customer, project,
   product, task category, symptom, environment, or follow-up relationship, each
   with a short reason and a relative Markdown link. Helps: surface prior fixes
   and context instead of starting cold.
   Diagram: Current case → related links → prior cases.

6. **Decision Brief** — on demand, generate a single sanitized,
   NotebookLM-optimized Markdown brief (Options, Diagnostic Tree, Risk/Rollback,
   Recommendation) in the case `Output/` folder; upload it to NotebookLM for a
   decision mind map. Helps: turn scattered case findings into a structured,
   visual decision aid.
   Diagram: Case memory → decision brief → NotebookLM mind map.

7. **Case Lifecycle** — manage status transitions (Active, Blocked, Closed,
   Reopened, Follow-up Created); move closed entries from `active-cases.md` to
   `closed-cases.md`; capture reusable closure summaries. Helps: an accurate,
   auditable picture of every case's state over time.
   Diagram: Active → Blocked/Closed → Reopened/Follow-up.

## Directory Structure Section

Two annotated ASCII trees, each folder/file with a one-line description.

### Case Workspace

```text
<root>/<customer>/<project>/<yyyymmdd>_<product_abbrev>_<short_case_name>/
  <case_directory>.md   # case-local memory: identity, problem, evidence, decisions, closure
  Reference/            # source docs, vendor notes, reference material
  Logs/                 # collected log files
  Docs/                 # customer or project documents
  Screenshots/          # captured screenshots
  Scripts/              # scripts used during the case
  Output/               # generated artifacts, including decision briefs
```

### Central Memory

```text
<root>/Caseflow/memories/
  workspace-index.md          # root workspace, coverage, and skill-gap notes
  active-cases.md             # index of active/blocked/reopened cases
  closed-cases.md             # permanent index of closed and follow-up-created cases
  customers/<customer>.md     # reusable per-customer knowledge
  projects/<customer>/<project>.md  # reusable per-project context and environment
  products/<product>.md       # reusable per-product context
  patterns/<task_category>.md # reusable per-task-category evidence needs and checklists
```

## Acceptance Criteria

- `caseflow/README.md` exists and starts with `# Caseflow`.
- Contains a `## Features` section with exactly 7 `###`-level feature entries,
  each having a **How it works** line, a **How it helps** line, and one fenced
  ```text ASCII diagram.
- Contains a `## Directory Structure` section with the two annotated trees
  (Case Workspace and Central Memory).
- Contains a `## How to Use` pointer to `SKILL.md`.
- Does not modify any other file.
- ASCII diagrams render correctly (no broken fences).
