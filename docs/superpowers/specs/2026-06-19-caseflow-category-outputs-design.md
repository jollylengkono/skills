# Caseflow Category Output Artifacts — Design

Date: 2026-06-19
Status: Approved (pending implementation plan)

## Goal

Add two category-routed output artifacts to the caseflow skill, generated into
the case `Output/` folder and linked from the case file. Caseflow selects the
artifact by the case's task category. The pattern mirrors the existing Decision
Brief feature.

## Routing

| Task category | Artifact |
|---|---|
| `troubleshooting`, `performance-tuning` | Findings Report |
| `installation-configuration`, `patching`, `upgrade` | Task & Blocker Log |

## Artifact 1: Findings Report

Email-ready, on-demand dated snapshots.

- Three parallel numbered sections, cross-referenced by item number:
  - `## Findings` — numbered list (1, 2, 3, …)
  - `## Analysis` — numbered list aligned to the finding numbers
  - `## Recommendations` — numbered list aligned to the finding numbers
- A short header block (case name, date, one-line summary) so it pastes cleanly
  into an email body.
- A banner stating the report is sanitized and cleared for sharing.
- Saved as `Output/<case_directory>_findings_<yyyymmdd>.md`. A new dated file is
  created each time a report is generated, preserving what each email contained.

## Artifact 2: Task & Blocker Log

A single living file per case, updated in place as work progresses.

- `## Tasks` table with columns: `# | Task | Status | Start | Done`.
  Start and Done are datetimes; Status is a short word (Pending, In progress,
  Done, Blocked).
- `## Blocker Records` table with columns:
  `# | Blocker | Owner | Raised | Needed by | Status | Resolution`.
- Saved as `Output/<case_directory>_tasks-blockers.md`. One file per case,
  updated in place (not re-dated) as tasks start/finish and blockers open/close.

## Architecture

Mirror the Decision Brief pattern.

- Create `caseflow/templates/findings-report.md` — fill-in template for the
  Findings Report.
- Create `caseflow/templates/task-blocker-log.md` — fill-in template for the
  Task & Blocker Log.
- Create `caseflow/references/category-outputs.md` — one combined reference that
  holds the shared rules (routing table, `Output/` location, secrets-only
  sanitization, case-file linking) plus a subsection per artifact. Combined
  rather than two files to keep the shared rules DRY.
- Add a concise `## Category Output Artifacts` section to `caseflow/SKILL.md`,
  with updates to the Directory Structure tree and Key Starting Points.

## Lifecycle and Trigger

- Both artifacts are generated on demand when the engineer asks.
- Findings Report: generate a new dated snapshot each time.
- Task & Blocker Log: create once, then update the same file in place over the
  life of the case.

## Sanitization

Secrets-only, reusing caseflow's existing rule: strip credentials, passwords,
wallets, private keys, certificates, and production connection strings; keep
customer name, hostnames, and versions. This matters especially for the Findings
Report, which is pasted into customer email. Each artifact carries a top banner
stating it is cleared for external sharing.

## Linking

Each artifact is linked from the case file under its Evidence / Outputs area,
using a relative Markdown link with a short label, following caseflow's
relative-link rules (include the `.md` filename).

## README Update

Add one new feature entry to `caseflow/README.md`: **Category Output Artifacts**,
following the existing per-feature template (How it works, How it helps, and one
pure-ASCII diagram). The diagram shows task category routing to the two
artifacts. This becomes the 8th feature in the README's Features section.

## Out of Scope (YAGNI)

- No auto-generation triggers at milestones or closure.
- No email sending or external integration.
- No output formats beyond Markdown.
- No changes to the existing Decision Brief feature.

## Acceptance Criteria

- `caseflow/templates/findings-report.md` exists with `## Findings`,
  `## Analysis`, `## Recommendations` sections and a sanitization banner.
- `caseflow/templates/task-blocker-log.md` exists with a `## Tasks` table
  (`# | Task | Status | Start | Done`) and a `## Blocker Records` table
  (`# | Blocker | Owner | Raised | Needed by | Status | Resolution`).
- `caseflow/references/category-outputs.md` exists with the routing table, both
  artifact workflows, the sanitization gate, and linking rules.
- `caseflow/SKILL.md` has a `## Category Output Artifacts` section, the two new
  files in its Directory Structure tree, and a Key Starting Points entry.
- `caseflow/README.md` has a new Category Output Artifacts feature entry with a
  pure-ASCII diagram.
- No secrets appear in any new file.
