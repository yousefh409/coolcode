---
name: pipeline-build
description: Build a feature with the full pipeline. Supports --spec, --plan, --resume, --no-qa, --type, --budget flags.
disable-model-invocation: true
user-invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Skill, Agent, AskUserQuestion
argument-hint: "[feature description] [--spec file|--plan file|--resume|--no-qa|--type bugfix|small|feature|refactor|hotfix|spike|dep-upgrade|security-audit|--budget N]"
---

# Pipeline Build

Full pipeline for building a feature on an existing project.

**Input:** `$ARGUMENTS` = feature description + optional flags

Pipeline state: !`cat .claude/pipeline-state.json 2>/dev/null || echo "NONE"`
GSD installed: !`which gsd 2>/dev/null && echo "YES" || echo "NO"`
gstack installed: !`ls ~/.claude/skills/gstack/review/SKILL.md 2>/dev/null && echo "YES" || echo "NO"`
Project docs: !`ls docs/*.md 2>/dev/null || echo "NO DOCS — run /pipeline-init first"`
Test strategy: !`[ -f .gsd/TESTING.md ] && head -5 .gsd/TESTING.md || echo "NO TESTING.md"`

## Argument Parsing

Parse `$ARGUMENTS` for flags (anything with `--` prefix). Everything else is the feature description.

| Flag | Effect |
|------|--------|
| `--spec <file>` | Skip to Phase 3 (plan), use provided spec |
| `--plan <file>` | Skip to Phase 4 (execute), use provided plan |
| `--resume` | Read pipeline-state.json, enter at last incomplete phase |
| `--no-qa` | Skip Phase 5 (QA) |
| `--type <type>` | Execution strategy (see routing table) |
| `--budget <N>` | Token budget for GSD execution |

**`--type` values:**
- `bugfix` — triage, fix, verify (default for bug descriptions)
- `small` — scope, plan, implement, verify (few tasks)
- `feature` — full pipeline (default)
- `refactor` — inventory, plan, migrate, verify
- `hotfix` — urgent P0, minimal process, fast path
- `spike` — research/prototype, no shipping expected
- `dep-upgrade` — dependency update with regression testing
- `security-audit` — security-focused audit and fix cycle

If no feature description found and no --resume/--spec/--plan, ask user what they want to build.

## Pre-flight Checks

1. Verify `docs/` exists. If not: "Run `/pipeline-init` first to set up pipeline infrastructure."
2. Verify `CLAUDE.md` exists. If not: same message.
3. If `--resume` and no `pipeline-state.json` exists: "Nothing to resume. Starting fresh."

## Resume Logic

If `--resume` or pipeline state exists:
1. Read `.claude/pipeline-state.json`
2. Find first phase where `status != "completed"`
3. `AskUserQuestion`: "Pipeline was interrupted at [phase name]. Resume or restart?"
4. Resume → enter at that phase with existing artifacts
5. Restart → delete state, start at Phase 1

If the interrupted phase was execution, check `artifacts.executionEngine` in pipeline state:
- If `"gsd"`: resume with `gsd --continue`
- If `"superpowers"`: resume in current session — Superpowers worktree and branch are still intact
- If not set: ask the user which engine was in use

## Phase 1: Requirements

### Initialize state

Write `.claude/pipeline-state.json`:
```json
{
  "version": 2,
  "pipeline": "build",
  "feature": "<description>",
  "startedAt": "<ISO-8601>",
  "flags": { "noQa": false, "type": "feature", "budget": null },
  "artifacts": { "executionEngine": null },
  "phases": {
    "1": { "name": "requirements", "status": "in_progress" },
    "2": { "name": "brainstorm-spec", "status": "pending" },
    "3": { "name": "plan", "status": "pending" },
    "3.5": { "name": "design-gate", "status": "pending" },
    "4": { "name": "execute", "status": "pending" },
    "5": { "name": "qa", "status": "pending" },
    "6": { "name": "doc-update", "status": "pending" },
    "7": { "name": "ship", "status": "pending" }
  }
}
```

### Gather requirements

**For `--type hotfix`:** Skip requirements gathering — go directly to Phase 2 or Phase 4. Hotfixes need speed.

**For `--type spike`:** Use `AskUserQuestion` — "What question should this spike answer?" and "What's the time box?" Then skip to Phase 4 (no spec, no plan — just explore).

If gstack installed:
```
Skill(gstack:office-hours)
```
Provide the feature description + project context (read CLAUDE.md, docs/ARCHITECTURE.md).

If gstack NOT installed, use `AskUserQuestion`:
1. "What exactly should this feature do?" (Header: "Feature")
2. "What are the edge cases or constraints?" (Header: "Constraints")
3. "How should it be tested?" (Header: "Testing")

Capture the pre-execution git ref:
```bash
git rev-parse HEAD
```
Store as `artifacts.baseRef`.

Update state: Phase 1 → `"completed"`, store `artifacts.requirementsPath`.

## Phase 2: Brainstorm + Spec

*Skip if `--spec` flag was provided — store the provided spec path and jump to Phase 3.*
*Skip for `--type hotfix` — go to Phase 4 with just the bug description.*
*Skip for `--type spike` — go to Phase 4.*

Load project context for the brainstorming session:
- Read `CLAUDE.md`, `docs/ARCHITECTURE.md`
- If `.gsd/KNOWLEDGE.md` exists, read it
- Read requirements from Phase 1

Invoke:
```
Skill(superpowers:brainstorming)
```

Superpowers brainstorming will:
1. Ask clarifying questions
2. Propose approaches with trade-offs
3. Present design sections for user approval
4. Write spec to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
5. Commit spec

Update state: Phase 2 → `"completed"`, store `artifacts.specPath`.

## Phase 3: Plan

*Skip if `--plan` flag was provided — store the provided plan path and jump to Phase 3.5.*

Superpowers brainstorming's terminal state invokes `writing-plans` automatically. Before it does, inject:

> **CRITICAL: Tag each task in the plan with exactly one of these tags on the task header line:**
> - `<!-- tag: ui-new -->` — brand new UI component (needs Paper design gate before implementation)
> - `<!-- tag: ui-update -->` — modification to existing UI component
> - `<!-- tag: logic -->` — backend, logic, infrastructure, configuration, tests
>
> Example: `- [ ] **Task 3: Build the settings panel** <!-- tag: ui-new -->`

If using `--spec`: invoke `Skill(superpowers:writing-plans)` explicitly with the spec file.

After the plan is written, writing-plans will offer an execution choice (subagent-driven vs inline). **DO NOT let execution start yet.** Tell the user the design gate check comes first.

1. Note the plan path
2. Update state: Phase 3 → `"completed"`, store `artifacts.planPath`
3. Proceed to design gate check

## Phase 3.5: Design Gate

Read the plan file. Grep for `<!-- tag: ui-new -->`.

If ui-new tasks found:
```
Skill(pipeline-design-gate) with plan file path
```

If none: skip.

Update state: Phase 3.5 → `"completed"`, store `artifacts.designArtifacts`.

## Phase 4: Execute

Read the plan to understand scope. The `--type` flag and task count determine execution strategy.

### Step 4a: Set up worktree (Superpowers path only)

**REQUIRED before Superpowers execution.** If routing table selects a Superpowers strategy, invoke:
```
Skill(superpowers:using-git-worktrees)
```
This creates an isolated branch so implementation doesn't wreck the current branch. It will:
- Create a worktree in `.worktrees/` (or ask user preference)
- Verify `.worktrees/` is in `.gitignore`
- Install dependencies in the worktree
- Run tests to establish a clean baseline

**Skip for GSD execution** — GSD creates its own branch automatically (e.g., `gsd/small-feature/...`).

### Step 4b: Execute via routing table

| --type | GSD installed | Strategy |
|--------|--------------|----------|
| `bugfix` | Yes | `gsd start bugfix` |
| `bugfix` | No | `Skill(superpowers:systematic-debugging)` → `Skill(superpowers:test-driven-development)` |
| `small` | Yes | `gsd start small-feature` |
| `small` | No | `Skill(superpowers:subagent-driven-development)` |
| `feature` (default) | Yes, >10 tasks | Offer choice: Superpowers (sequential) or `gsd start full-project` (parallel) |
| `feature` (default) | Yes, ≤10 tasks | `Skill(superpowers:subagent-driven-development)` |
| `feature` (default) | No | `Skill(superpowers:subagent-driven-development)` |
| `refactor` | Yes | `gsd start refactor` |
| `refactor` | No | `Skill(superpowers:subagent-driven-development)` |
| `hotfix` | Yes | `gsd start hotfix` |
| `hotfix` | No | `Skill(superpowers:test-driven-development)` directly |
| `spike` | Yes | `gsd start spike` |
| `spike` | No | Explore and prototype in current session — no subagent needed |
| `dep-upgrade` | Yes | `gsd start dep-upgrade` |
| `dep-upgrade` | No | `Skill(superpowers:subagent-driven-development)` |
| `security-audit` | Yes | `gsd start security-audit` |
| `security-audit` | No | `Skill(gstack:cso)` for audit, then TDD for fixes |

#### For Superpowers subagent-driven-development:

Store `artifacts.executionEngine = "superpowers"` in pipeline state.

Invoke `Skill(superpowers:subagent-driven-development)` with:
- The full plan text (don't make subagents read the file)
- For `<!-- tag: ui-new -->` tasks: include `.gsd/designs/task-{N}-{name}.md` content

Superpowers handles: dispatch per task → spec compliance review → code quality review → TDD → commits. Each task gets a fresh subagent.

After execution, Superpowers auto-invokes `finishing-a-development-branch`. **Let it complete** — it presents 4 options (merge locally, create PR, keep branch, discard). Do NOT interrupt.

#### For GSD execution:

Store `artifacts.executionEngine = "gsd"` in pipeline state.

If `--budget` flag: configure GSD preferences first.

For `--type` workflows (bugfix, hotfix, refactor, spike, dep-upgrade, security-audit):
```bash
gsd headless "start <template> <description>" --no-session
```

For `feature` with >10 tasks (full project mode):
```bash
gsd headless "start full-project <feature description>" --no-session
```

GSD creates its own milestone, plans tasks, and executes in parallel waves.

Monitor progress:
```bash
gsd headless "query"  # ~50ms, no LLM, returns JSON
```

GSD runs until complete. Check exit: 0 = complete, 1 = error.

**After GSD completes**, explicitly invoke finishing-a-development-branch (GSD does NOT call it automatically):
```
Skill(superpowers:finishing-a-development-branch)
```

### Step 4c: Finish the development branch

`finishing-a-development-branch` presents 4 options:
1. **Merge back locally** — merges worktree branch into base
2. **Push and create PR** — pushes branch and creates PR via `gh`
3. **Keep branch as-is** — leaves worktree intact for later
4. **Discard** — deletes branch and work

Note which option the user chose. Store in state as `artifacts.branchOutcome`.

Update state: Phase 4 → `"completed"`.

## Phase 5: QA

*Skip if `--no-qa` flag.*
*Skip if `--type spike` — spikes don't get QA.*

Invoke: `Skill(pipeline-qa) --skip-review`

The `--skip-review` flag prevents double review — `gstack:ship` in Phase 7 runs `gstack:review` internally.

pipeline-qa reads `.gsd/TESTING.md` and dispatches checks. It uses `gstack:qa-only` for browser testing (report only, no fixes) and routes any fixes through proper channels.

Update state: Phase 5 → `"completed"`, store `artifacts.qaReportPath`.

## Phase 6: Doc Update

*Skip if `--type spike`.*

Invoke: `Skill(pipeline-doc-update)` with `artifacts.baseRef`

This runs in a fork — reads git diff, updates docs/, commits.

Update state: Phase 6 → `"completed"`.

## Phase 7: Ship + Cleanup

*Skip if `--type spike`.*

### Ship options

The right ship action depends on what happened in Phase 4:

**If user chose "Push and create PR" in finishing-a-development-branch:**
→ PR already exists. Offer gstack:ship enhancements:

`AskUserQuestion`: "PR already created. Want the full ship workflow?" (Header: "Ship")
- "Yes — VERSION bump, CHANGELOG, review" → `Skill(gstack:ship)` — this updates the existing PR
- "No — PR is good as-is" → skip

**If user chose "Merge back locally":**
→ Code is on the base branch. Offer shipping:

`AskUserQuestion`: "Code merged locally. What's next?" (Header: "Ship")
- "Full ship (VERSION, CHANGELOG, PR)" → `Skill(gstack:ship)`
- "Push to remote" → `git push`
- "Done" → skip

**If user chose "Keep branch" or GSD was used:**
→ Work is on a branch. Offer full ship:

`AskUserQuestion`: "Ready to ship?" (Header: "Ship")
- "Full ship (VERSION, CHANGELOG, PR)" → `Skill(gstack:ship)`
- "Just push the branch" → `git push -u origin HEAD`
- "Done — I'll handle it later" → skip

### Deploy chain (if ship created a PR)

If `gstack:ship` was used and created/updated a PR, offer the deploy chain:

`AskUserQuestion`: "PR is up. Want to land and deploy?" (Header: "Deploy")
- "Yes — merge, wait for CI, verify production" → `Skill(gstack:land-and-deploy)` — this merges the PR, waits for CI/deploy, then runs `Skill(gstack:canary)` automatically
- "No — I'll merge manually" → skip

### Cleanup

1. Delete `.claude/pipeline-state.json`
2. If gstack learn available: `Skill(gstack:learn)` to store learnings from this build
3. Final commit if uncommitted changes

Report: "Feature built, tested, and documented."
