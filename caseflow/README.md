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
  +----------+   creates
  | Caseflow |-----------> <customer>/<project>/<date>_<prod>_<case>/
  +----------+               |
                             +-- <case_directory>.md   (from templates/case.md)
                             +-- Reference/ Logs/ Docs/ Screenshots/ Scripts/ Output/
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

**How it works:** Case-local `<case_directory>.md` records what happened on this
specific case. Central `Caseflow/memories/` records reusable knowledge organized by
customer, project, product, and task pattern. Both are linked with relative
Markdown links.

**How it helps:** You keep full per-case detail and still build cross-case
knowledge you can reuse, without copy-pasting between cases.

```text
  +--------------------+   relative links   +--------------------------+
  | Case-local memory  |<------------------>| Central memory           |
  | (what happened)    |                    | customers/  projects/    |
  +--------------------+                    | products/   patterns/    |
                                            | (reusable knowledge)     |
                                            +--------------------------+
```

### 4. Product Skill Routing

**How it works:** Caseflow maps the Oracle product (and, for Fusion and OEM, the
task category) to the matching local product skill — `db`, `apex`, `oci`, `oem`,
`graal`, or a `fusion` sub-skill. If no skill exists, it falls back to the closest domain
skill and records the gap.

**How it helps:** The right product playbook loads automatically, so you start
with relevant guidance instead of generic steps.

```text
  Oracle product + task category
         |
         v
  +---------------+
  | routing table |--> db/  apex/  oci/  oem/  graal/  fusion/{weblogic,ohs,soa,...}
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
  Active <--> Blocked        (pause / resume)
    |           |
    +-----+-----+
          v
       Closed --> Reopened          (same issue returns)
          |
          +-----> Follow-up Created (new related scope)
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
