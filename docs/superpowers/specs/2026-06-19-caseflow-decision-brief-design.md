# Caseflow Decision Brief — Design

Date: 2026-06-19
Status: Approved (pending implementation plan)

## Goal

Add an on-demand caseflow sub-workflow that produces a single, focused,
NotebookLM-optimized Markdown document for a specific decision. The engineer
uploads that document to NotebookLM, which renders the decision mind map.

Division of responsibility:

- **Caseflow** owns the *source document* (structured, grounded, sanitized).
- **NotebookLM** owns the *visualization* (the mind map), generated manually
  after upload.

There is no programmatic call to NotebookLM. The handoff is a manual file upload.

## Scope

In scope:

- A new template `caseflow/templates/decision-brief.md`.
- A new concise "Decision Brief" section in `caseflow/SKILL.md`.
- A new `caseflow/references/decision-brief.md` holding the detailed workflow.
- Updates to the Directory Structure tree and Key Starting Points in `SKILL.md`.
- Linking generated briefs from the case file's existing `## Decisions` section.

Out of scope (YAGNI — may be added later):

- NotebookLM API integration / automated mind-map generation.
- Reimporting the generated mind map back into the case folder.
- Auto-triggering brief generation at milestones or at closure.
- Identifier redaction beyond secrets (customer name and hostnames are kept).

## Output

A single Markdown file — the decision brief, which doubles as the NotebookLM
source. No other artifact is produced by caseflow; the mind map itself is
created inside NotebookLM after the engineer uploads this file.

One decision per file. Multiple briefs may coexist as a case evolves.

## Where It Is Generated

Inside the case's existing `Output/` folder, using caseflow's case-named
convention.

Filename pattern:

```text
<case_directory>_decision_<slug>_<yyyymmdd>.md
```

- `<slug>` is a 1–3 word kebab-case label for the decision (e.g.
  `patch-vs-upgrade`).
- `<yyyymmdd>` is the local current date.

Example for case directory `20260527_db_listener_issue`:

```text
<root>/<customer>/<project>/20260527_db_listener_issue/
├── 20260527_db_listener_issue.md        # case file (link added under ## Decisions)
├── Output/
│   └── 20260527_db_listener_issue_decision_patch-vs-upgrade_20260619.md
├── Reference/  Logs/  Docs/  Screenshots/  Scripts/
```

## Template Structure

The heading hierarchy *is* the shape of the NotebookLM mind map, so the template
defines the branches explicitly.

- **Title (H1)** = the decision question. This becomes the central node.
- **## Context** — short self-contained grounding (problem, environment, current
  state) so the map is not ungrounded. Pulled from the case file.
- **## Options & Trade-offs** — one `###` per candidate option, each with pros,
  cons, effort, and risk.
- **## Diagnostic Decision Tree** — symptom → hypothesis → test → branch logic as
  nested bullets.
- **## Risk & Rollback Gates** — go/no-go conditions, blast radius, rollback
  triggers, maintenance-window constraints.
- **## Recommendation & Rationale** — recommended path, supporting evidence,
  rejected alternatives.
- **## How to use in NotebookLM** — a short footer: upload this file as a source
  and open Mind Map.

A one-line banner at the top states the brief is sanitized and cleared for
external upload.

## Workflow (On-Demand)

Triggered when the engineer explicitly asks for a decision brief during an active
case.

1. Confirm the decision question and a short `<slug>`.
2. Read the existing case file and pull relevant Context, Findings, and Decisions.
3. Populate `templates/decision-brief.md` with the four decision branches.
4. Run the sanitization gate (see below).
5. Save the artifact to the case `Output/` folder using the filename pattern.
6. Add a one-line link to the brief under the case file's `## Decisions` section,
   using a relative Markdown link with a short reason (per caseflow link rules).
7. Report the generated path and remind the engineer to upload it to NotebookLM.

## Sanitization Gate

Reuse caseflow's existing secrets rule. Strip credentials, passwords, wallets,
private keys, certificates, and production connection strings. Keep customer
name, hostnames, and versions — these are needed for a useful decision map.

The brief is explicitly export-bound (it leaves for an external Google service),
so the sanitization step is mandatory before saving, and the brief states at the
top that it has been cleared for external upload.

## Linking and Caseflow Conventions

- The generated brief is an output artifact, stored under `Output/`.
- The case file's `## Decisions` section gets a relative Markdown link to each
  brief with a short reason.
- All links follow caseflow's existing relative-link rules and include the `.md`
  filename.
- No secrets are stored in the brief or in any memory file.

## SKILL.md Changes

- Add the `templates/decision-brief.md` and `references/decision-brief.md` entries
  to the Directory Structure tree.
- Add a concise "Decision Brief" section describing the on-demand trigger and
  pointing to `references/decision-brief.md` for detail.
- Add a Decision Brief entry to Key Starting Points.
