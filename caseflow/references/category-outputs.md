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
