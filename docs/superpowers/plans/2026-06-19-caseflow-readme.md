# Caseflow README Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create `caseflow/README.md`, a human-facing overview with a 7-feature list (each with how-it-works, how-it-helps, and a pure-ASCII architecture diagram) and an annotated directory-structure section.

**Architecture:** Single new Markdown file. All content is fully specified in this plan. Verification is structural (`grep` checks for required headings, balanced code fences, and the per-feature how-it-works/how-it-helps lines). No code, no test framework.

**Tech Stack:** Markdown only. Diagrams are plain ASCII inside fenced ```text blocks (portable on GitHub, terminal, and any editor). Repo of record: `/home/jolly_lengkono/skills`, branch `docs/caseflow-readme`.

Spec: `docs/superpowers/specs/2026-06-19-caseflow-readme-design.md`.

---

## File Structure

- Create: `caseflow/README.md` — the human-facing overview. One responsibility: explain what caseflow does (features) and what it creates on disk (directory structure), pointing to `SKILL.md` as the authoritative behavior spec.

**Note on deployment:** The live runtime copy of the skill lives at `~/.claude/skills/caseflow/`. A README is human-facing documentation that the skill runtime does not load, so syncing it is optional. Task 2 mirrors it for consistency; skip Task 2 if you do not want the live copy updated.

---

### Task 1: Create caseflow/README.md

**Files:**
- Create: `caseflow/README.md`

- [ ] **Step 1: Confirm the file does not yet exist**

Run:
```bash
cd /home/jolly_lengkono/skills
test -f caseflow/README.md && echo EXISTS || echo ABSENT
```
Expected: ABSENT.

- [ ] **Step 2: Create `caseflow/README.md` with EXACTLY this content**

Write the file so its literal content is everything between the two `=== FILE CONTENT ===` guide lines below (do not include the guide lines themselves). The file's first line is `# Caseflow`. The content contains fenced ```text blocks — preserve them exactly, including the ASCII diagrams.

=== FILE CONTENT BEGIN ===
# Caseflow

Caseflow is the intake and memory workflow for Oracle Advanced Services
engagements. It turns ad-hoc customer work into a consistent, searchable case
workspace: it scaffolds customer/project/case folders, keeps both case-local and
central memory, routes you to the right Oracle product skill, links related
cases, and produces decision and closure artifacts.

`SKILL.md` is the authoritative behavior spec. This README is the human-facing
overview of what Caseflow does and what it creates on disk.

## Features

### 1. Case Intake & Workspace Scaffolding

**How it works:** You provide customer, project, task category, Oracle product,
and a short case name. Caseflow creates a dated, named case folder with standard
evidence subfolders and seeds case-local memory from a template.

**How it helps:** Every case lands in the same predictable place with the same
structure, so anyone can find evidence and resume the work months later without
guessing.

```text
  Engineer
     |  intake (customer, project, category, product, case name)
     v
  +----------+   creates   +-------------------------------------------+
  | Caseflow |------------>| <customer>/<project>/<date>_<prod>_<case>/ |
  +----------+             |   <case>.md   (from templates/case.md)     |
                           |   Reference/ Logs/ Docs/ Screenshots/      |
                           |   Scripts/ Output/                         |
                           +-------------------------------------------+
```

### 2. Directory Resolution

**How it works:** Before asking for paths, Caseflow walks upward from the current
directory to find `Caseflow/memories/workspace-index.md`, then resolves the root
workspace, customer, and project — asking only when a match is ambiguous or
missing.

**How it helps:** You can run Caseflow from anywhere inside the workspace without
re-entering the root path each time.

```text
  current dir
     |  walk up
     v
  .../Caseflow/memories/workspace-index.md  --> root workspace
                                                  |
                                                  +--> match customer/
                                                  +--> match project/
```

### 3. Two-Tier Memory

**How it works:** Case-local `<case>.md` records what happened on this specific
case. Central `Caseflow/memories/` records reusable knowledge organized by
customer, project, product, and task pattern. Both are linked with relative
Markdown links.

**How it helps:** You keep full per-case detail and still build cross-case
knowledge you can reuse, without copy-pasting between cases.

```text
  +--------------------+   relative links   +--------------------------+
  | Case-local memory  |<------------------>| Central memory           |
  | <case>.md          |                    | customers/  projects/    |
  | (what happened)    |                    | products/   patterns/    |
  +--------------------+                    | (reusable knowledge)     |
                                            +--------------------------+
```

### 4. Product Skill Routing

**How it works:** Caseflow maps the Oracle product (and, for Fusion and OEM, the
task category) to the matching local product skill — `db`, `apex`, `oci`, `oem`,
or a `fusion` sub-skill. If no skill exists, it falls back to the closest domain
skill and records the gap.

**How it helps:** The right product playbook loads automatically, so you start
with relevant guidance instead of generic steps.

```text
  Oracle product + task category
         |
         v
  +---------------+
  | routing table |--> db/  apex/  oci/  oem/  fusion/{weblogic,ohs,soa,...}
  +---------------+
         |
         +-- no match --> closest domain skill + record skill gap
```

### 5. Related Cases Linking

**How it works:** Caseflow links cases that share a customer, project, product,
task category, symptom, environment, or follow-up relationship. Each link carries
a short reason and a relative Markdown link, kept selective and high-signal.

**How it helps:** Prior fixes, checklists, and risks surface automatically, so
you reuse hard-won context instead of starting cold.

```text
        +---------------+
        | Current case  |
        +-------+-------+
                | related links (with reasons)
    +-----------+-----------+
    v           v           v
  same        same        similar
  product     project     symptom
  case        case        case
```

### 6. Decision Brief

**How it works:** On demand, Caseflow generates a single sanitized,
NotebookLM-optimized Markdown brief — Context, Options & Trade-offs, Diagnostic
Decision Tree, Risk & Rollback Gates, and Recommendation — into the case
`Output/` folder. You upload it to NotebookLM to render a decision mind map.

**How it helps:** Scattered case findings become a structured, visual decision
aid you can think through and share for sign-off.

```text
  Case memory --> Caseflow --> Output/<case>_decision_<slug>_<date>.md
                                       |  upload
                                       v
                                  NotebookLM --> decision mind map
```

### 7. Case Lifecycle

**How it works:** Caseflow tracks case status — Active, Blocked, Closed,
Reopened, Follow-up Created — moving closed entries out of `active-cases.md` into
`closed-cases.md` and capturing reusable closure summaries.

**How it helps:** You get an accurate, auditable picture of every case's state
over time, with closure knowledge preserved for reuse.

```text
  Active --> Blocked --> Active
    |
    v
  Closed --> Reopened          (same issue returns)
    |
    +-------> Follow-up Created (new related scope)
```

## Directory Structure

Caseflow creates two trees under the root workspace: per-case folders and a
shared central-memory tree.

### Case Workspace

```text
<root>/<customer>/<project>/<yyyymmdd>_<product_abbrev>_<short_case_name>/
  <case_directory>.md   case-local memory: identity, problem, evidence, decisions, closure
  Reference/            source docs, vendor notes, reference material
  Logs/                 collected log files
  Docs/                 customer or project documents
  Screenshots/          captured screenshots
  Scripts/              scripts used during the case
  Output/               generated artifacts, including decision briefs
```

### Central Memory

```text
<root>/Caseflow/memories/
  workspace-index.md                 root workspace, product coverage, and skill-gap notes
  active-cases.md                    index of active, blocked, and reopened cases
  closed-cases.md                    permanent index of closed and follow-up-created cases
  customers/<customer>.md            reusable per-customer knowledge
  projects/<customer>/<project>.md   reusable per-project context and environment
  products/<product>.md              reusable per-product context
  patterns/<task_category>.md        reusable per-task-category evidence needs and checklists
```

## How to Use

Invoke the Caseflow skill before starting, continuing, or closing an Oracle
customer case. Caseflow handles intake, folder creation, memory updates, product
routing, and closure. For the complete behavior — required inputs, naming rules,
relative-link rules, routing tables, and the closure quality gate — see
[`SKILL.md`](SKILL.md).
=== FILE CONTENT END ===

- [ ] **Step 3: Run structural checks (all must pass)**

Run each and confirm the expected count:
```bash
cd /home/jolly_lengkono/skills
head -1 caseflow/README.md                                      # expect: # Caseflow
grep -c -E '^### [1-7]\. ' caseflow/README.md                   # expect: 7
grep -c -E '^\*\*How it works:\*\*' caseflow/README.md          # expect: 7
grep -c -E '^\*\*How it helps:\*\*' caseflow/README.md          # expect: 7
grep -c -E '^## (Features|Directory Structure|How to Use)$' caseflow/README.md   # expect: 3
grep -c -E '^### (Case Workspace|Central Memory)$' caseflow/README.md            # expect: 2
grep -c '^```' caseflow/README.md                               # expect: 18 (9 fenced blocks, balanced)
```
Expected outputs in order: `# Caseflow`, then `7`, `7`, `7`, `3`, `2`, `18`.

- [ ] **Step 4: Commit**

```bash
cd /home/jolly_lengkono/skills
git add caseflow/README.md
git commit -m "docs(caseflow): add README with feature overview and directory structure

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 2: Mirror the README to the live skills copy (optional)

Skip this task if your environment syncs `~/.claude/skills/` automatically, or if you do not want the live copy updated.

**Files:**
- Modify: `~/.claude/skills/caseflow/README.md` (live runtime copy)

- [ ] **Step 1: Copy the README into the live copy**

Run:
```bash
cp /home/jolly_lengkono/skills/caseflow/README.md /home/jolly_lengkono/.claude/skills/caseflow/README.md
```
Expected: no output (success).

- [ ] **Step 2: Verify the live README matches the repo**

Run:
```bash
diff /home/jolly_lengkono/skills/caseflow/README.md /home/jolly_lengkono/.claude/skills/caseflow/README.md && echo "README IN SYNC"
```
Expected: `README IN SYNC` (no diff). No commit — the live copy is outside the git repo.

---

## Final Verification

- [ ] `caseflow/README.md` exists and passes all structural checks in Task 1, Step 3.
- [ ] The README modifies no other file (`git show --stat HEAD` shows only `caseflow/README.md`).
- [ ] `git status --short` is clean (no stray files).
- [ ] All ASCII diagrams render correctly with balanced ```text fences.
