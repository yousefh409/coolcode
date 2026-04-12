---
name: pipeline-new
description: Start a brand new project from scratch. Full pipeline — scaffold, test setup, requirements, brainstorm, spec, plan, design, execute, QA, docs.
disable-model-invocation: true
user-invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Skill, Agent, AskUserQuestion
argument-hint: "[project name or description]"
---

# Pipeline New

Full pipeline for starting a new project from scratch.

**Input:** `$ARGUMENTS` = project name or description

Pipeline state: !`cat .claude/pipeline-state.json 2>/dev/null || echo "NONE"`
GSD installed: !`which gsd 2>/dev/null && echo "YES ($(gsd --version 2>/dev/null))" || echo "NO"`
gstack installed: !`ls ~/.claude/skills/gstack/review/SKILL.md 2>/dev/null && echo "YES" || echo "NO"`
Superpowers: !`ls ~/.claude/plugins/cache/claude-plugins-official/superpowers/*/skills/brainstorming/SKILL.md 2>/dev/null && echo "YES" || echo "NO"`

## Resume Logic

If pipeline state shows an incomplete phase:
1. Read state, find first phase where `status != "completed"`
2. Use `AskUserQuestion` — "Pipeline was interrupted at [phase name]. Resume or restart?"
3. Resume → skip completed phases, enter at incomplete phase
4. Restart → delete `.claude/pipeline-state.json`, start at Phase 0

If resuming at the execution phase, check `artifacts.executionEngine` in pipeline state:
- If `"gsd"`: resume with `gsd --continue`
- If `"superpowers"`: resume in current session — Superpowers worktree and branch are still intact
- If not set: ask the user which engine was in use

If no state exists, start at Phase 0.

## Phase 0: Scaffold + Test Setup

### 0a. Initialize pipeline state

Use the Write tool to create `.claude/pipeline-state.json`:
```json
{
  "version": 2,
  "pipeline": "new",
  "feature": "$ARGUMENTS",
  "startedAt": "<ISO-8601>",
  "artifacts": { "executionEngine": null },
  "phases": {
    "0": { "name": "scaffold", "status": "in_progress" },
    "1": { "name": "requirements", "status": "pending" },
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

### 0b. Scaffold

Invoke: `Skill(pipeline-scaffold)` with `$ARGUMENTS`

This creates: git repo, docs/ stubs, CLAUDE.md, VERSION, .gsd/ directory, initializes GSD if installed.

Capture the initial commit SHA and store in `artifacts.baseRef`.

### 0c. Test Setup

Invoke: `Skill(pipeline-test-setup)`

This runs the interactive test strategy questionnaire and writes `.gsd/TESTING.md`.

Update state: Phase 0 → `"completed"`, store `artifacts.testStrategy: ".gsd/TESTING.md"`.

## Phase 1: Requirements

### If gstack installed:

Invoke: `Skill(gstack:office-hours)` with the project description from `$ARGUMENTS`.

Office hours will ask six forcing questions (demand reality, status quo, desperate specificity, narrowest wedge, observation, future-fit), challenge premises, and produce a design doc. It saves output to `~/.gstack/projects/{slug}/office-hours/`.

### If gstack NOT installed:

Use `AskUserQuestion` to gather requirements yourself:

1. "What problem does this solve and for whom?" (Header: "Problem")
2. "What are the 3-5 core features for MVP?" (Header: "MVP scope")
3. "What are the technical constraints?" (Header: "Constraints", options: specific framework, specific platform, existing API to integrate, performance requirements, Other)
4. "What does success look like?" (Header: "Success")

Write answers to `.gsd/REQUIREMENTS.md` using the Write tool.

Update state: Phase 1 → `"completed"`, store `artifacts.requirementsPath`.

## Phase 2: Brainstorm + Spec

Load project context for the brainstorming session:
- Read `CLAUDE.md`, `docs/ARCHITECTURE.md`
- If `.gsd/KNOWLEDGE.md` exists, read it
- Read requirements from Phase 1

Invoke:
```
Skill(superpowers:brainstorming)
```

Provide as context:
- Requirements from Phase 1 (read the requirements file)
- Project CLAUDE.md
- Any existing codebase context

Superpowers brainstorming will:
1. Ask clarifying questions (one at a time)
2. Propose 2-3 approaches with trade-offs
3. Present design in sections with user approval
4. Write spec to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
5. Commit the spec to git

**DO NOT skip this or try to auto-answer.** The brainstorming skill is interactive — it needs the user.

After spec is committed, update state: Phase 2 → `"completed"`, store `artifacts.specPath`.

## Phase 3: Plan

Superpowers brainstorming's terminal state invokes `writing-plans` automatically. Before it does, inject this instruction:

> **CRITICAL: Tag each task in the plan with exactly one of these tags on the task header line:**
> - `<!-- tag: ui-new -->` — brand new UI component (needs Paper design gate before implementation)
> - `<!-- tag: ui-update -->` — modification to existing UI component
> - `<!-- tag: logic -->` — backend, logic, infrastructure, configuration, tests
>
> Example: `- [ ] **Task 3: Build the user profile card** <!-- tag: ui-new -->`

The writing-plans skill will:
1. Create a detailed plan with checkbox steps, exact file paths, complete code blocks
2. Save to `docs/superpowers/plans/YYYY-MM-DD-<feature>.md`
3. Offer an execution choice (subagent-driven vs inline)

**DO NOT let execution start yet.** After the plan is written:
1. Note the plan path
2. Update state: Phase 3 → `"completed"`, store `artifacts.planPath`
3. Tell the user: "Plan is ready. Checking for UI components that need design before execution."
4. Proceed to design gate check

## Phase 3.5: Design Gate

Read the plan file. Grep for `<!-- tag: ui-new -->`.

**If ui-new tasks exist:**

Invoke: `Skill(pipeline-design-gate)` with the plan file path as argument.

This creates 3 Paper design options per component, user picks one, exports are saved to `.gsd/designs/`.

After completion, read pipeline state to get design artifact paths.

**If NO ui-new tasks:**

Skip. Update state: Phase 3.5 → `"completed"` with note "skipped — no ui-new tasks".

## Phase 4: Execute

Read the plan to count tasks and understand scope.

### Step 4a: Set up worktree

**REQUIRED before Superpowers execution.** If using Superpowers path, invoke:
```
Skill(superpowers:using-git-worktrees)
```
This creates an isolated branch so implementation doesn't wreck the base branch. It will:
- Create a worktree in `.worktrees/` (or ask user preference)
- Verify `.worktrees/` is in `.gitignore`
- Install dependencies in the worktree
- Run tests to establish a clean baseline

**Skip for GSD execution** — GSD creates its own branch automatically.

### Step 4b: Execute

**Default path: Superpowers subagent-driven-development**

Store `artifacts.executionEngine = "superpowers"` in pipeline state.

Invoke: `Skill(superpowers:subagent-driven-development)`

Provide:
- The full plan text (don't make subagents read the file)
- For `<!-- tag: ui-new -->` tasks: include the corresponding `.gsd/designs/task-{N}.md` content so the implementer uses the approved design

Superpowers will dispatch a fresh subagent per task with two-stage review (spec compliance → code quality). It handles TDD, commits, and review loops.

After execution, Superpowers auto-invokes `finishing-a-development-branch`. **Let it complete** — it presents 4 options (merge locally, create PR, keep branch, discard). Do NOT interrupt.

**Alternative: GSD autonomous (for large projects, >10 tasks)**

If GSD is installed and the plan has >10 tasks, offer:

`AskUserQuestion`: "This plan has N tasks. Use Superpowers (sequential, reviewed) or GSD (parallel, autonomous)?"
- "Superpowers (Recommended)" — sequential with two-stage review per task
- "GSD autonomous" — parallel waves, faster but less granular review

If GSD chosen:

Store `artifacts.executionEngine = "gsd"` in pipeline state.

```bash
gsd headless "start full-project <feature description>" --no-session
```

GSD creates its own milestone, plans tasks, and executes in parallel waves.

Monitor: `gsd headless "query"` (instant JSON, ~50ms).
GSD runs until complete. Check exit: 0 = complete, 1 = error.

**After GSD completes**, explicitly invoke finishing-a-development-branch (GSD does NOT call it automatically):
```
Skill(superpowers:finishing-a-development-branch)
```

### Step 4c: Finish the development branch

`finishing-a-development-branch` presents 4 options: merge locally, create PR, keep branch, or discard.

Store the user's choice in `artifacts.branchOutcome`.

Update state: Phase 4 → `"completed"`.

## Phase 5: QA

Invoke: `Skill(pipeline-qa) --skip-review`

The `--skip-review` flag prevents double review — `gstack:ship` in Phase 7 runs `gstack:review` internally.

This reads `.gsd/TESTING.md` and dispatches checks. Uses `gstack:qa-only` for browser testing (report only), routes fixes through proper debugging channels.

If critical issues found → pipeline-qa will handle fix routing and re-verification.

Update state: Phase 5 → `"completed"`, store `artifacts.qaReportPath`.

## Phase 6: Doc Update

Invoke: `Skill(pipeline-doc-update)` with the base ref from `artifacts.baseRef`.

This runs in a fork — reads git diff, updates docs/ files, commits.

Update state: Phase 6 → `"completed"`.

## Phase 7: Ship + Cleanup

### Ship options

The right ship action depends on what happened in Phase 4:

**If user chose "Push and create PR" in finishing-a-development-branch:**
→ PR already exists. Offer gstack:ship enhancements:

`AskUserQuestion`: "PR already created. Want the full ship workflow?" (Header: "Ship")
- "Yes — VERSION bump, CHANGELOG, review" → `Skill(gstack:ship)` — updates the existing PR
- "No — PR is good as-is" → skip

**If user chose "Merge back locally":**
→ Code is on base branch. Offer shipping:

`AskUserQuestion`: "Code merged locally. What's next?" (Header: "Ship")
- "Full ship (VERSION, CHANGELOG, PR)" → `Skill(gstack:ship)`
- "Push to remote" → `git push`
- "Done" → skip

**If user chose "Keep branch" or GSD was used:**

`AskUserQuestion`: "Ready to ship?" (Header: "Ship")
- "Full ship (VERSION, CHANGELOG, PR)" → `Skill(gstack:ship)`
- "Just push the branch" → `git push -u origin HEAD`
- "Done — I'll handle it later" → skip

### Deploy chain (if ship created a PR)

If `gstack:ship` was used and a PR exists, offer:

`AskUserQuestion`: "PR is up. Want to land and deploy?" (Header: "Deploy")
- "Yes — merge, wait for CI, verify production" → `Skill(gstack:land-and-deploy)`
- "No — I'll merge manually" → skip

### Cleanup

1. Delete `.claude/pipeline-state.json`
2. If gstack learn is available, invoke `Skill(gstack:learn)` to review and store learnings
3. Final commit if there are uncommitted changes

Report: "Pipeline complete. Project scaffolded, tested, and documented."
