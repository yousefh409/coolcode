---
name: pipeline-build
description: Build a feature end-to-end. Brainstorm → plan → execute (Superpowers or GSD) → update docs → QA → fix until clean → ship.
disable-model-invocation: true
user-invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Skill, Agent, AskUserQuestion
argument-hint: "<feature description>"
---

# Pipeline Build

End-to-end feature build. Each step delegates to the right tool — this skill only orchestrates.

**Input:** `$ARGUMENTS` = feature description

GSD installed: !`which gsd 2>/dev/null && echo "YES" || echo "NO"`
gstack installed: !`ls ~/.claude/skills/gstack/review/SKILL.md 2>/dev/null && echo "YES" || echo "NO"`
Project docs: !`[ -f .gsd/PROJECT.md ] && head -5 .gsd/PROJECT.md || echo "NO PROJECT.md"`
Test strategy: !`[ -f .gsd/TESTING.md ] && head -5 .gsd/TESTING.md || echo "NO TESTING.md"`

## Process

### 1. Ground

Read the project context before doing anything:

```
Read(.gsd/PROJECT.md)
Read(.gsd/KNOWLEDGE.md)
Read(.gsd/RUNTIME.md)
Read(CLAUDE.md)
```

If `.gsd/PROJECT.md` doesn't exist, run `/pipeline-init` first:
```
Skill(pipeline-init)
```

### 2. Brainstorm

```
Skill(superpowers:brainstorming)
```

Pass the feature description from `$ARGUMENTS`. Brainstorming outputs a spec to `docs/superpowers/specs/` and auto-chains into `writing-plans` which outputs a plan to `docs/superpowers/plans/`.

Wait for both to complete. Note the spec file path — GSD needs it if the user chooses the GSD path.

### 3. Choose Execution Engine

Use `AskUserQuestion`:
- Header: "Execution"
- Question: "How should this be executed?"
- Options:
  - "Superpowers" — description: "Focused execution with worktree isolation. Best for interactive work, TDD-style development, and tasks you want to steer."
  - "GSD autonomous" — description: "Parallel wave execution with crash recovery. Best for large tasks (>10 subtasks), hands-off execution."

### 4. Execute

**If Superpowers:**
```
Skill(superpowers:subagent-driven-development)
```
Superpowers auto-manages worktree setup. After execution:
```
Skill(superpowers:finishing-a-development-branch)
```
This presents merge/PR/keep/discard options.

**If GSD:**
```bash
gsd headless new-milestone --context <spec-file-path> --auto
```
Where `<spec-file-path>` is the spec from Step 2 (e.g., `docs/superpowers/specs/2024-01-15-feature-name.md`).

GSD manages its own branch, crash recovery, verification enforcement, and updates `.gsd/` docs (PROJECT.md, KNOWLEDGE.md, RUNTIME.md, DECISIONS.md) automatically during milestone execution.

### 5. Update Docs

After execution completes, sync documentation:

**If GSD was used:** GSD already updated `.gsd/` docs during the milestone. Run doc-update as a safety net:
```
Skill(pipeline-doc-update)
```

**If Superpowers was used:** This step is essential since GSD's docs are stale:
```bash
gsd headless "init" --no-session
```
This refreshes `.gsd/CODEBASE.md` (the file map). Then:
```
Skill(pipeline-doc-update)
```
This reads the git diff and updates `.gsd/PROJECT.md`, `.gsd/RUNTIME.md`, `.gsd/KNOWLEDGE.md`, and appends to `.gsd/DECISIONS.md`.

### 6. QA + Iterate

Run QA:
```
Skill(pipeline-qa)
```

If QA finds critical or important issues:

1. **Present ALL findings** — show the complete QA report with every issue, severity, and file path
2. **Ask qualifying questions** using `AskUserQuestion`:
   - Header: "Fix approach"
   - Question: "QA found N issues. How should I approach fixes?"
   - Options:
     - "Fix all" — description: "Fix every critical and important issue, then re-run QA"
     - "Fix critical only" — description: "Fix only critical issues, leave important as known"
     - "Let me review first" — description: "I'll look at the report and tell you what to fix"
3. **Auto-fix** each issue using the appropriate tool:
   - Bug with unclear cause → `Skill(superpowers:systematic-debugging)`
   - Bug with clear fix → `Skill(superpowers:test-driven-development)`
   - Browser UI bug → `Skill(gstack:qa)`
   - Security vulnerability → `Skill(superpowers:test-driven-development)` (write test proving vuln, then fix)
4. **Re-run QA** after all fixes:
   ```
   Skill(pipeline-qa)
   ```
5. **Repeat** steps 1-4 until QA is clean (no critical or important issues).

### 7. Ship

When QA passes, suggest shipping:

Use `AskUserQuestion`:
- Header: "Ship"
- Question: "QA is clean. Ready to ship?"
- Options:
  - "Ship it" — description: "VERSION bump, CHANGELOG, review, PR"
  - "Not yet" — description: "I want to make more changes first"

If ship:
```
Skill(gstack:ship)
```

## Done

Report:
- What was built (link to spec)
- Execution path used (Superpowers or GSD)
- QA results (final pass)
- Number of fix iterations
- Ship status (PR link if shipped)
