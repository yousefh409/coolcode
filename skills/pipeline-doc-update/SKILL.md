---
name: pipeline-doc-update
description: Update .gsd/ project documentation from recent code changes. Reads git diff, updates PROJECT.md, RUNTIME.md, KNOWLEDGE.md, appends to DECISIONS.md.
disable-model-invocation: true
user-invocable: true
context: fork
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Pipeline Doc Update

Update `.gsd/` docs to reflect recent code changes. Concise targeted edits only — no prose, no filler.

**Input:** `$ARGUMENTS` = git base ref to diff against (commit SHA, branch name, or `HEAD~N`)

GSD installed: !`which gsd 2>/dev/null && echo "YES" || echo "NO"`

## Scope

**Update:** `.gsd/PROJECT.md`, `.gsd/RUNTIME.md`, `.gsd/KNOWLEDGE.md`, `CLAUDE.md` (index only)
**Append only:** `.gsd/DECISIONS.md`
**Refresh:** Run `gsd headless "init"` to regenerate `.gsd/CODEBASE.md` (if GSD installed)
**Never touch:** `.gsd/STATE.md`, `.gsd/M###-*`, `.gsd/S##-*`, `.gsd/T##-*`, `.gsd/TESTING.md`, `docs/superpowers/*`

## Process

### 1. Refresh CODEBASE.md

```bash
gsd headless "init" --no-session 2>/dev/null || true
```

### 2. Get changed files

```bash
git diff --name-only $ARGUMENTS..HEAD
```

Fallback: `git diff --name-only HEAD~5`. If no changes, exit.

### 3. Categorize and update

Read changed files. Map to target docs:

| Change type | Target |
|------------|--------|
| Architecture (new dirs, entry points, services) | `.gsd/PROJECT.md` |
| API (routes, handlers, middleware) | `.gsd/RUNTIME.md` |
| Components / Models | `.gsd/PROJECT.md` |
| Patterns, conventions | `.gsd/KNOWLEDGE.md` |
| Architectural decisions | `.gsd/DECISIONS.md` (append) |

Skip: test files, config tweaks, lock files, build output, `.gsd/` state files.

Read the target doc, then apply **concise** targeted edits:

- **Modified feature:** Edit existing section in place
- **New feature:** Insert after the relevant section
- **Deleted feature:** Remove the section
- **New decision:** Append `## YYYY-MM-DD — Title` entry to DECISIONS.md

If target doc doesn't exist, create it with a heading and the content.

**Writing rules:**
- Key facts only — file paths, relationships, behavior. No explanatory prose.
- Point to code, don't duplicate it
- Match existing doc formatting
- Preserve GSD-written content

### 4. Commit

```bash
git add .gsd/PROJECT.md .gsd/RUNTIME.md .gsd/KNOWLEDGE.md .gsd/DECISIONS.md CLAUDE.md 2>/dev/null
git diff --cached --quiet || git commit -m "docs: update from recent changes"
```

## Done

Report which docs were updated and what changed.
