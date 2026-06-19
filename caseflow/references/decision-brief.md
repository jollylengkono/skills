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
3. Copy `templates/decision-brief.md` and populate every branch:
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
   (see the `## How to use in NotebookLM` section in the generated file).

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
