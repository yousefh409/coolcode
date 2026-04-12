---
name: pipeline-quick
description: Quick fix or small feature. Routes to the right Superpowers skill based on task type. Conditional doc update.
disable-model-invocation: true
user-invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Skill, Agent, AskUserQuestion
argument-hint: "[description of fix or small feature] [--careful|--no-docs]"
---

# Pipeline Quick

Lightweight pipeline for small fixes and minor features. No GSD, no design gate, no pipeline state file.

**Input:** `$ARGUMENTS` = description + optional flags

Project context:
!`[ -f CLAUDE.md ] && head -50 CLAUDE.md || echo "No CLAUDE.md — run /pipeline-init first"`
!`[ -f .gsd/TESTING.md ] && head -10 .gsd/TESTING.md || echo "No TESTING.md"`
gstack installed: !`ls ~/.claude/skills/gstack/review/SKILL.md 2>/dev/null && echo "YES" || echo "NO"`

## Argument Parsing

Parse `$ARGUMENTS` for flags:

| Flag | Effect |
|------|--------|
| `--careful` | Enable gstack careful mode for production-touching changes |
| `--no-docs` | Skip conditional doc update |

Everything without `--` prefix is the fix/feature description.

If no description provided, use `AskUserQuestion`:
- "What do you want to fix or add?" (Header: "Task")
- Options: free text (user will select "Other")

## Process

### 1. Classify the Task

Read the description and project context. Route to the correct tool:

**Bug with error message / stack trace / production issue:**
→ Go to Step 2A (Debugging)

**Bug with clear cause** — you know exactly what's wrong and where (typo, wrong config, off-by-one):
→ Go to Step 2B (TDD Fix)

**Small feature or unclear scope** — needs design decisions, multiple approaches:
→ Go to Step 2C (Brainstorm)

### 2A. Bug — Debugging Path

If description mentions production, deploy, or live system AND gstack installed:
```
Skill(gstack:investigate)
```
gstack investigate has access to the browse daemon for inspecting live apps and uses the learnings framework to check for previously seen patterns. It follows 4 phases: investigate → analyze → hypothesize → implement. Iron law: no fixes without root cause.

Otherwise:
```
Skill(superpowers:systematic-debugging)
```
Superpowers systematic-debugging follows 4 mandatory phases:
1. Root cause investigation (read errors, reproduce, trace data flow)
2. Pattern analysis (find working examples, compare)
3. Hypothesis and testing (single hypothesis, minimal change)
4. Implementation (failing test first, then single fix)

After the debugging skill completes, go to Step 3.

### 2B. Clear Fix — TDD Path

If `--careful` flag OR description mentions production/deploy/migration:
```
Skill(gstack:careful)
```
This enables safety guardrails for destructive operations.

```
Skill(superpowers:test-driven-development)
```
Provide the fix description and relevant file context. TDD will:
1. Write a failing test proving the bug exists
2. Verify the test fails for the right reason
3. Write minimal code to make it pass
4. Verify the test passes and no others break
5. Refactor if needed

After TDD completes, go to Step 3.

### 2C. Small Feature — Brainstorm Path

```
Skill(superpowers:brainstorming)
```
Provide the description + project context (CLAUDE.md, ARCHITECTURE.md summaries).

Brainstorming will:
1. Ask clarifying questions
2. Propose 2-3 approaches
3. Write a spec to `docs/superpowers/specs/`
4. Auto-invoke `writing-plans`
5. Offer execution choice → `subagent-driven-development` or `executing-plans`

Let the full Superpowers chain complete naturally (brainstorm → plan → worktree → execute → finish branch).

Go to Step 3.

### 3. Verify

```
Skill(superpowers:verification-before-completion)
```
This runs the project's test command (from CLAUDE.md, `.gsd/TESTING.md`, or package.json) with fresh output. It refuses to claim success without evidence.

### 4. Conditional Doc Update

*Skip if `--no-docs` flag.*

Check what changed:
```bash
git diff --name-only HEAD~1
```

**If structural changes** (new files, new directories, new APIs, new components, schema changes, new routes):
→ Invoke `Skill(pipeline-doc-update)` with `HEAD~1`

**If cosmetic changes** (bug fixes, typo fixes, style tweaks within existing files):
→ Skip doc update.

## Done

Report what was fixed/added. No cleanup needed — no pipeline state was created.
