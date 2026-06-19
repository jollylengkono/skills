# Caseflow Portable Memory Design

## Context

Caseflow currently documents central memory under `<root_workspace_path>/.codex/memories/` and creates case folders with the date suffix:

```text
<root_workspace_path>/<customer>/<project>/<product_abbrev>_<short_case_name>_<yyyymmdd>/
```

The requested update is to make Caseflow ownership clearer by using a skill-specific central memory directory and to make case folders sort naturally by date.

## Goals

- Store Caseflow central memory under `<root_workspace_path>/.caseflow/memories/`.
- Create new case folders with the date prefix: `<yyyymmdd>_<product_abbrev>_<short_case_name>`.
- Prefer root-relative paths in persisted memory files so cases remain portable across workstations, mount points, usernames, and WSL path casing.
- Preserve enough root-discovery behavior that Caseflow can be used while the shell is inside an existing case directory.

## Non-Goals

- Migrate existing `.codex/memories` data automatically.
- Rename existing case folders.
- Change product routing, task categories, evidence subfolders, or case template sections except where path wording needs to be clearer.

## Path Strategy

Caseflow should distinguish runtime paths from persisted references.

Runtime operations need a resolved root workspace path. The skill should still ask for a full root workspace path during new-case intake unless it can safely discover one from the current directory.

Persisted memory should use root-relative paths for case links and reusable references. For example:

```text
Mandiri/New MCM/20260523_jvm_heap_full_gc
```

The absolute root may still appear in a workspace identity field where useful, but it should not be the primary link format in central memory.

## Root Discovery

When continuing a case or when the engineer is already working inside a case directory, Caseflow should locate central memory by walking upward from the current directory until it finds:

```text
.caseflow/memories/workspace-index.md
```

If found, the parent directory containing `.caseflow/` is the root workspace path. If not found, Caseflow asks for the full root workspace path and uses normal directory resolution.

## Case Folder Naming

New case folders should use:

```text
<root_workspace_path>/<customer>/<project>/<yyyymmdd>_<product_abbrev>_<short_case_name>/
```

Rules:

- Use the local current date in `yyyymmdd` format.
- Lowercase the product abbreviation and short case name.
- Replace spaces in short case name with underscores.
- Preserve existing customer and project directory spelling.

Example:

```text
Mandiri/New MCM/20260523_jvm_heap_full_gc/
```

## Central Memory Layout

The new central memory layout is:

```text
<root_workspace_path>/.caseflow/memories/
  workspace-index.md
  active-cases.md
  customers/<customer>.md
  projects/<customer>/<project>.md
  products/<product>.md
  patterns/<task_category>.md
```

All Caseflow instructions that currently mention `<root_workspace_path>/.codex/memories/` should be changed to `<root_workspace_path>/.caseflow/memories/`.

The missing-product-skill fallback should record skill gaps under:

```text
<root_workspace_path>/.caseflow/memories/workspace-index.md
```

Closure updates should target the same `.caseflow/memories` files.

## Case-Local Memory

`case.md` should keep the existing identity and evidence sections. Add a `Case Relative Path:` identity field so the case-local file remains portable even if `Case Path:` contains an absolute path.

Recommended identity fields:

```text
- Case Path:
- Case Relative Path:
```

Central memory should link to the relative path. Case-local memory may record both absolute and relative paths.

## Error Handling

- If root discovery finds multiple possible roots only through ambiguous user input, ask the engineer to choose.
- If no `.caseflow/memories/workspace-index.md` is found when continuing from a case directory, ask for the full root workspace path.
- If `.codex/memories` exists but `.caseflow/memories` does not, do not silently migrate. Ask whether to create `.caseflow/memories` for new Caseflow memory.

## Testing

Validate the spec update with targeted checks:

- Search `caseflow` for remaining `.codex/memories` references.
- Search for the old case folder pattern `<product_abbrev>_<short_case_name>_<yyyymmdd>`.
- Confirm the new `.caseflow/memories` path appears consistently in setup, missing-skill fallback, common mistakes, and closure sections.
- Confirm the case template includes `Case Relative Path:`.

## Implementation Scope

The implementation should update `/home/jolly_lengkono/skills/caseflow/SKILL.md` and `/home/jolly_lengkono/skills/caseflow/templates/case.md`.

After source updates, refresh or reinstall the installed skill copy if the local skill workflow requires it.
