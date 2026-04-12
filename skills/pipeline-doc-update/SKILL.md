---
name: pipeline-doc-update
description: Update project documentation from recent code changes. Reads git diff, updates docs/ files.
disable-model-invocation: true
user-invocable: false
context: fork
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Pipeline Doc Update

Update living documentation to reflect recent code changes. Runs in a forked context to avoid polluting the main session.

**Input:** `$ARGUMENTS` = git base ref to diff against (commit SHA, branch name, or `HEAD~N`)

## Scope Rules

**DO update these files:**
- `docs/ARCHITECTURE.md` — system design, data flow, key directories
- `docs/API.md` — endpoints, params, responses
- `docs/COMPONENTS.md` — UI component inventory
- `docs/MODELS.md` — data shapes, types, relationships
- `CLAUDE.md` — index updates only (new sections, changed tech stack)

**DO NOT touch these files (other tools manage them):**
- `docs/superpowers/specs/*` — Superpowers manages specs
- `docs/superpowers/plans/*` — Superpowers manages plans
- `.gsd/*` (except `.gsd/designs/`) — GSD manages its own state
- `.gsd/TESTING.md` — pipeline-test-setup manages this
- `Brand.md` — brand-kit skill manages this

## Process

### 1. Get changed files

```bash
git diff --name-only $ARGUMENTS..HEAD
```

If the base ref doesn't exist, try:
```bash
git diff --name-only HEAD~5
```

If no changes found, report "No changes since base ref" and exit.

### 2. Read changed files and categorize

Read each changed file (use the Read tool, not cat). Categorize:

| Category | Indicators |
|----------|------------|
| Architecture | New directories, new services, changed entry points, new config files, moved files |
| API | Files in routes/, api/, endpoints/; changed HTTP handlers; new/modified middleware |
| Components | Files in components/, pages/, screens/, views/; new .tsx/.vue/.svelte files |
| Models | Files in models/, types/, schemas/; migration files; changed interfaces/types |

Skip files that don't affect documentation:
- Test files (*.test.*, *.spec.*)
- Config tweaks (tsconfig, eslint, prettier)
- Lock files (package-lock, bun.lock, yarn.lock)
- Build output
- `.gsd/` files (GSD manages its own docs)

### 3. Read current docs

For each affected category, read the corresponding doc file using the Read tool:
- Architecture changes → read `docs/ARCHITECTURE.md`
- API changes → read `docs/API.md`
- Component changes → read `docs/COMPONENTS.md`
- Model changes → read `docs/MODELS.md`

### 4. Apply targeted edits

For each affected doc, use the **Edit tool** with `old_string`/`new_string`:

**For modified features:** Find the existing section and edit it in place.
```
Edit(file_path="docs/API.md", old_string="existing section content", new_string="updated section content")
```

**For new features:** Find the right location and insert after it.
```
Edit(file_path="docs/API.md", old_string="## Existing Section\n\ncontent", new_string="## Existing Section\n\ncontent\n\n## New Section\n\nnew content")
```

**For deleted features:** Remove the section.
```
Edit(file_path="docs/COMPONENTS.md", old_string="## Deleted Component\n\nold content\n\n", new_string="")
```

**Rules:**
- Reference actual file paths (e.g., `src/components/Button.tsx:15-42`)
- Don't duplicate code in docs — point to it
- Each section should be scannable in 10 seconds
- If grep can find it in under 5 seconds, don't document it
- Match the formatting style already in the doc

### 5. Update CLAUDE.md index

Only if structural changes warrant it (new major sections, changed tech stack). Use the Edit tool.

### 6. Commit

```bash
git add docs/ CLAUDE.md
git diff --cached --quiet || git commit -m "docs: update from recent changes"
```

The `diff --cached --quiet` check prevents empty commits.

## Done

Report which docs were updated and what changed. The caller handles routing.
