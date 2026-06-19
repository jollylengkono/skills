# Caseflow Portable Memory Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Update the Caseflow skill source so central memory uses `.caseflow/memories`, new case folders are date-prefixed, and persisted case links prefer root-relative paths.

**Architecture:** This is a documentation-driven skill implementation. The behavior is encoded in `caseflow/SKILL.md`, while the case-local identity fields are encoded in `caseflow/templates/case.md`. No migration automation or existing case renaming is included.

**Tech Stack:** Markdown skill source, shell verification with `rg`, no build system.

---

## File Structure

- Modify: `/home/jolly_lengkono/skills/caseflow/SKILL.md`
  - Owns Caseflow workflow instructions, directory resolution, case naming, central memory layout, root discovery, related-case links, missing-skill fallback, mistakes, and closure paths.
- Modify: `/home/jolly_lengkono/skills/caseflow/templates/case.md`
  - Adds `Case Relative Path:` to local case identity.
- Do not modify: `/home/jolly_lengkono/.agents/skills/caseflow/*`
  - Installed skill copy is outside the requested source repo update. Refresh separately only if the engineer requests it.
- Do not stage or commit: `/home/jolly_lengkono/skills/docs/superpowers/*`
  - User explicitly requested not to track the superpowers docs directory.

---

### Task 1: Update Central Memory Path Wording

**Files:**
- Modify: `/home/jolly_lengkono/skills/caseflow/SKILL.md`

- [ ] **Step 1: Replace the setup path**

Change the "How to Use This Skill" central memory step from:

```markdown
4. Create or update central memory under `<root_workspace_path>/.codex/memories/`.
```

to:

```markdown
4. Create or update central memory under `<root_workspace_path>/.caseflow/memories/`.
```

- [ ] **Step 2: Replace the Central Memory layout path**

Change:

```markdown
<root_workspace_path>/.codex/memories/
```

to:

```markdown
<root_workspace_path>/.caseflow/memories/
```

inside the "Central Memory" section.

- [ ] **Step 3: Replace missing-skill fallback path**

Change:

```markdown
4. Record the missing product skill under a `Skill Gaps` section in `<root_workspace_path>/.codex/memories/workspace-index.md`.
```

to:

```markdown
4. Record the missing product skill under a `Skill Gaps` section in `<root_workspace_path>/.caseflow/memories/workspace-index.md`.
```

- [ ] **Step 4: Replace common mistake path**

Change:

```markdown
- Writing central memory under the shell's current directory instead of `<root_workspace_path>/.codex/memories/`.
```

to:

```markdown
- Writing central memory under the shell's current directory instead of `<root_workspace_path>/.caseflow/memories/`.
```

- [ ] **Step 5: Replace closure paths**

Change the closure bullets from:

```markdown
- `<root_workspace_path>/.codex/memories/customers/<customer>.md`.
- `<root_workspace_path>/.codex/memories/projects/<customer>/<project>.md`.
- `<root_workspace_path>/.codex/memories/products/<product>.md`.
- `<root_workspace_path>/.codex/memories/patterns/<task_category>.md`.
- `<root_workspace_path>/.codex/memories/active-cases.md`.
```

to:

```markdown
- `<root_workspace_path>/.caseflow/memories/customers/<customer>.md`.
- `<root_workspace_path>/.caseflow/memories/projects/<customer>/<project>.md`.
- `<root_workspace_path>/.caseflow/memories/products/<product>.md`.
- `<root_workspace_path>/.caseflow/memories/patterns/<task_category>.md`.
- `<root_workspace_path>/.caseflow/memories/active-cases.md`.
```

- [ ] **Step 6: Verify no active `.codex/memories` instruction remains**

Run:

```bash
rg -n "\.codex/memories" /home/jolly_lengkono/skills/caseflow
```

Expected: no output.

---

### Task 2: Update Case Folder Naming Convention

**Files:**
- Modify: `/home/jolly_lengkono/skills/caseflow/SKILL.md`

- [ ] **Step 1: Replace case folder pattern**

Change:

```markdown
<root_workspace_path>/<customer>/<project>/<product_abbrev>_<short_case_name>_<yyyymmdd>/
```

to:

```markdown
<root_workspace_path>/<customer>/<project>/<yyyymmdd>_<product_abbrev>_<short_case_name>/
```

- [ ] **Step 2: Clarify naming rules**

Change:

```markdown
Use underscores in generated case folder names. Lowercase product abbreviation and short case name. Preserve existing customer and project directory spelling.
```

to:

```markdown
Use the local current date in `yyyymmdd` format. Use underscores in generated case folder names. Lowercase product abbreviation and short case name. Preserve existing customer and project directory spelling.
```

- [ ] **Step 3: Verify old folder pattern is gone**

Run:

```bash
rg -n "<product_abbrev>_<short_case_name>_<yyyymmdd>" /home/jolly_lengkono/skills/caseflow
```

Expected: no output.

- [ ] **Step 4: Verify new folder pattern exists**

Run:

```bash
rg -n "<yyyymmdd>_<product_abbrev>_<short_case_name>" /home/jolly_lengkono/skills/caseflow/SKILL.md
```

Expected: one match in the Case Structure section.

---

### Task 3: Add Root Discovery and Relative Path Rules

**Files:**
- Modify: `/home/jolly_lengkono/skills/caseflow/SKILL.md`

- [ ] **Step 1: Extend Directory Resolution**

Insert this block after "Confirm the root workspace path exists." in the "Directory Resolution" section:

```markdown
2. When continuing work from inside an existing case directory, walk upward from the current directory until `.caseflow/memories/workspace-index.md` is found.
3. If found, use the parent directory containing `.caseflow/` as the root workspace path.
4. If not found, ask the engineer for the full root workspace path.
```

Then renumber the remaining directory-resolution steps so customer and project matching still happen after root resolution.

- [ ] **Step 2: Add persisted path guidance**

Insert this paragraph after the central memory layout block:

```markdown
Persist root-relative paths in central memory case links whenever possible, for example `Mandiri/New MCM/20260523_jvm_heap_full_gc`. Use absolute paths for runtime filesystem operations, but do not make absolute workstation-specific paths the primary link format in reusable central memory.
```

- [ ] **Step 3: Add no-silent-migration rule**

Insert this paragraph in the Central Memory section after the path guidance:

```markdown
If `<root_workspace_path>/.codex/memories/` exists but `<root_workspace_path>/.caseflow/memories/` does not, do not silently migrate existing memory. Ask whether to create `.caseflow/memories/` for new Caseflow memory or whether the engineer wants a separate migration step.
```

- [ ] **Step 4: Verify root discovery wording exists**

Run:

```bash
rg -n "walk upward|\\.caseflow/memories/workspace-index\\.md|root-relative paths|silently migrate" /home/jolly_lengkono/skills/caseflow/SKILL.md
```

Expected: matches for root discovery, relative links, and no-silent-migration guidance.

---

### Task 4: Add Case Relative Path to Template

**Files:**
- Modify: `/home/jolly_lengkono/skills/caseflow/templates/case.md`

- [ ] **Step 1: Add identity field**

Change:

```markdown
- Case Path:
- Task Category:
```

to:

```markdown
- Case Path:
- Case Relative Path:
- Task Category:
```

- [ ] **Step 2: Verify template field exists**

Run:

```bash
rg -n "Case Relative Path" /home/jolly_lengkono/skills/caseflow/templates/case.md
```

Expected: one match in the Identity section.

---

### Task 5: Final Verification

**Files:**
- Read: `/home/jolly_lengkono/skills/caseflow/SKILL.md`
- Read: `/home/jolly_lengkono/skills/caseflow/templates/case.md`

- [ ] **Step 1: Review Caseflow source around changed sections**

Run:

```bash
sed -n '1,240p' /home/jolly_lengkono/skills/caseflow/SKILL.md
```

Expected: central memory setup, directory resolution, case structure, central memory, missing-skill fallback, common mistakes, and closure sections are internally consistent.

- [ ] **Step 2: Review case template identity fields**

Run:

```bash
sed -n '1,40p' /home/jolly_lengkono/skills/caseflow/templates/case.md
```

Expected: Identity includes both `Case Path:` and `Case Relative Path:`.

- [ ] **Step 3: Run targeted consistency checks**

Run:

```bash
rg -n "\.codex/memories|<product_abbrev>_<short_case_name>_<yyyymmdd>" /home/jolly_lengkono/skills/caseflow
```

Expected: no output.

Run:

```bash
rg -n "\.caseflow/memories|<yyyymmdd>_<product_abbrev>_<short_case_name>|Case Relative Path|root-relative paths" /home/jolly_lengkono/skills/caseflow
```

Expected: matches for all new concepts.

- [ ] **Step 4: Check git status without staging**

Run:

```bash
git -C /home/jolly_lengkono/skills status --short
```

Expected: modified Caseflow files are visible; `docs/` remains untracked unless the engineer later chooses to track it.

---

## Execution Notes

- Do not commit or stage the superpowers docs directory.
- Do not migrate existing workspace memory from `.codex/memories` to `.caseflow/memories`.
- Do not rename existing case directories.
- Do not update the installed `~/.agents/skills/caseflow` copy unless explicitly requested after source changes.
