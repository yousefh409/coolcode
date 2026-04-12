# Pipeline Skills Full Rewrite Plan

## Why

The current skills are thin orchestration wrappers that:
- Fight GSD by creating `.gsd/` files GSD manages itself
- Assume Superpowers "auto-activates" (it doesn't — needs explicit `Skill()` calls)
- Don't know what files each tool creates or expects
- Have a static TESTING.md template instead of interactive setup
- Lack error handling, state recovery, and proper fallbacks
- Don't leverage GSD's workflow templates (bugfix, small-feature, etc.)
- Miss gstack's expectations (skill routing in CLAUDE.md, browse daemon, VERSION/CHANGELOG)

## Tool Contracts (What We Must Respect)

### Superpowers
- **brainstorming** → outputs `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` + git commit
- **writing-plans** → outputs `docs/superpowers/plans/YYYY-MM-DD-<feature>.md` (NO placeholders, TDD embedded, checkbox steps)
- **subagent-driven-development** → sequential per-task: implementer → spec-reviewer → code-quality-reviewer. NEVER parallel implementers
- **test-driven-development** → red-green-refactor, mandatory fail verification
- **verification-before-completion** → fresh command output before ANY success claim
- **using-superpowers** → check skills BEFORE any response. Invoke via `Skill()` tool, never Read
- **Invocation:** ALL via `Skill(superpowers:<name>)`. They do NOT auto-activate without the using-superpowers meta-skill

### GSD v2 CLI
- `gsd init` → creates `.gsd/` with ~20 files (PROJECT.md, REQUIREMENTS.md, KNOWLEDGE.md, DECISIONS.md, STATE.md, roadmap.md, PREFERENCES.md, etc.)
- `gsd headless auto` → autonomous execution of current milestone
- `gsd headless query` → instant JSON state snapshot (~50ms, no LLM)
- `gsd headless next` → execute one unit (cron-friendly)
- `gsd headless dispatch <phase>` → force a specific phase
- `.gsd/PREFERENCES.md` → model routing, git config, execution modes, budget
- Workflow templates: `full-project`, `bugfix`, `small-feature`, `refactor`, `spike`, `hotfix`, `security-audit`
- **Do NOT create .gsd/ files manually** — let `gsd init` or `gsd` handle it

### gstack
- `/office-hours` → interactive requirements, saves design doc
- `/review` → analyzes diff against base branch
- `/qa` → headless browser test + fix loop
- `/qa-only` → report only, no fixes
- `/cso` → OWASP + STRIDE security audit
- `/browse` → headless Chromium daemon (~100ms/command)
- `/benchmark` → Core Web Vitals, page load, bundle size baselines
- `/ship` → test, review, VERSION bump, CHANGELOG, push, PR
- `/land-and-deploy` → merge PR, wait CI, verify production
- `/canary` → post-deploy health monitoring
- `/learn` → persistent learnings management
- `/careful` → destructive command warnings

## Architecture Changes

### 1. Stop creating .gsd/ files
`pipeline-scaffold` and `pipeline-init` currently create `.gsd/KNOWLEDGE.md`, `.gsd/REQUIREMENTS.md` etc.
**Change:** Delegate to `gsd init` when GSD is installed. Only create our own files (TESTING.md, designs/) if GSD isn't present.

### 2. Explicit Superpowers invocations
Replace every "let Superpowers auto-activate" with exact `Skill(superpowers:brainstorming)`, `Skill(superpowers:writing-plans)`, etc.

### 3. Interactive TESTING.md
Replace the static template with an interactive questionnaire using `AskUserQuestion`:
- Platform (web/mobile/desktop/api/cli/library)
- Test framework + exact run command
- Setup commands (dev server, DB seed, simulator boot)
- Environment variables needed
- Deployment target (for browser QA)
- CI system (for gstack:ship integration)

### 4. Use GSD workflow templates for pipeline-build sizing
Instead of ≤5/>5 task threshold:
- Bug fix → `gsd start bugfix`
- Small feature → `gsd start small-feature`
- Large feature → `gsd start full-project` or Superpowers subagent-driven-development
- Refactor → `gsd start refactor`

### 5. Proper state management
Pipeline state tracks phase completion + artifact paths. Each phase records what it produced:
```json
{
  "specPath": "docs/superpowers/specs/2026-04-12-feature-design.md",
  "planPath": "docs/superpowers/plans/2026-04-12-feature.md",
  "designArtifacts": { "task-1": ".gsd/designs/task-1-component.md" },
  "baseRef": "abc1234",
  "testStrategy": ".gsd/TESTING.md"
}
```

### 6. Add task tagging instruction to writing-plans invocation
When invoking `Skill(superpowers:writing-plans)`, prepend instruction:
> Tag each task header with `<!-- tag: ui-new -->`, `<!-- tag: ui-update -->`, or `<!-- tag: logic -->`

This gets injected as context, not by modifying Superpowers' skill.

## Skills to Rewrite (9 total)

### Skill 1: pipeline-scaffold (helper)
**Changes:**
- Remove all `.gsd/` file creation (KNOWLEDGE.md, REQUIREMENTS.md)
- Run `gsd init` if installed — it creates everything
- Only create: `docs/` stubs, `CLAUDE.md` (from template), `.gsd/designs/`, `.gitignore`
- Add gstack skill routing section to generated CLAUDE.md
- Add `VERSION` file (for gstack:ship)

### Skill 2: pipeline-test-setup (NEW helper)
**Purpose:** Interactive test strategy questionnaire. Called by pipeline-init and pipeline-new.
**Process:**
1. `AskUserQuestion` with 4 questions:
   - Platform + framework
   - Test runner + command
   - What test types apply (multi-select: unit, integration, e2e-browser, e2e-simulator, visual, perf, security)
   - CI system
2. Follow-up questions based on selections:
   - If e2e-browser: dev server command, deployed URL
   - If e2e-simulator: simulator type, test command (maestro/detox/xctest)
   - If integration: setup commands, env vars needed, DB seed
   - For any: teardown commands, auth/secrets needed
3. Write `.gsd/TESTING.md` from answers
4. Add testing summary to `CLAUDE.md`

### Skill 3: pipeline-doc-update (helper)
**Changes:**
- Add awareness of Superpowers' `docs/superpowers/specs/` and `docs/superpowers/plans/`
- Add awareness of GSD's `.gsd/` files (don't document what GSD tracks)
- Scope: only update `docs/ARCHITECTURE.md`, `docs/API.md`, `docs/COMPONENTS.md`, `docs/MODELS.md`
- Explicit Edit tool usage with old_str/new_str
- For new sections: use Write with append

### Skill 4: pipeline-design-gate (helper)
**Changes:**
- Remove reference to non-existent `/brand-kit` — check if `Brand.md` exists at project root
- Add `AskUserQuestion` for user to pick option (not just "present and ask")
- Specify `.gsd/designs/task-{N}-{slug}.md` format explicitly (markdown with JSX code block + style tokens)
- Add error handling for Paper MCP failures (skip gracefully)
- When no Brand.md exists, generate design brief from project context instead of suggesting a non-pipeline skill

### Skill 5: pipeline-new (entry point)
**Changes:**
- Phase 0 (Scaffold): invoke `Skill(pipeline-scaffold)` then `Skill(pipeline-test-setup)`
- Phase 1 (Requirements): explicit `Skill(gstack:office-hours)` invocation, fallback to inline AskUserQuestion
- Phase 2 (Brainstorm + Spec): explicit `Skill(superpowers:brainstorming)` — this produces spec AND hands off to writing-plans
- Phase 3 (Plan): Superpowers handles this as terminal state of brainstorming. We inject tag instruction as context
- Phase 3.5 (Design Gate): conditional `Skill(pipeline-design-gate)` if ui-new tags found
- Phase 4 (Execute): Use GSD workflow template selection based on task count/type, OR Superpowers subagent-driven-development
- Phase 5 (QA): explicit `Skill(pipeline-qa)` invocation
- Phase 6 (Doc Update): explicit `Skill(pipeline-doc-update)` with base ref
- Phase 7 (Ship): offer `Skill(gstack:ship)` for PR creation
- Add: state tracks artifact paths, base ref captured before execution
- Add: resume logic reads state, finds first incomplete phase, offers resume/restart
- Add: error handling — if any phase fails, state shows failure reason, user can --resume

### Skill 6: pipeline-init (entry point)
**Changes:**
- Discovery: use Agent(Explore) subagent type (it exists as a built-in agent type)
- Don't create .gsd/ files: run `gsd init` or `gsd` to let it auto-detect stack
- Create docs/ from discovery (ARCHITECTURE, API, COMPONENTS, MODELS)
- Generate CLAUDE.md with gstack skill routing section
- Run `Skill(pipeline-test-setup)` for interactive test strategy
- Invoke `Skill(gstack:learn)` to seed learnings from discovered patterns
- Explicit Write tool for all file creation
- Commit with proper message

### Skill 7: pipeline-build (entry point)
**Changes:**
- Flag parsing: explicit rules for --spec, --plan, --resume, --no-qa, --budget, --type
- Add --type flag: `bugfix`, `small-feature`, `feature`, `refactor` (maps to GSD workflow templates)
- Phase 1 (Requirements): `Skill(gstack:office-hours)` — skip if --spec or --plan provided
- Phase 2 (Brainstorm + Spec + Plan): `Skill(superpowers:brainstorming)` with project context. Brainstorming produces spec → invokes writing-plans → produces plan. We inject tag instruction
- Phase 2.5 (Design Gate): conditional `Skill(pipeline-design-gate)`
- Phase 3 (Execute): GSD workflow template OR Superpowers subagent-driven-development
  - If --type bugfix: `gsd start bugfix`
  - If --type small-feature: `gsd start small-feature`
  - If --type feature (or default for many tasks): Superpowers subagent-driven-development (respects TDD, reviews)
  - If --type refactor: `gsd start refactor`
  - Fallback: Superpowers if GSD not installed
- Phase 4 (QA): `Skill(pipeline-qa)` unless --no-qa
- Phase 5 (Doc Update): `Skill(pipeline-doc-update)` with base ref
- Phase 6 (Ship): offer `Skill(gstack:ship)`
- State management: capture specPath, planPath, baseRef, designArtifacts
- Resume: read state, skip completed phases, re-enter at failure point

### Skill 8: pipeline-quick (entry point)
**Changes:**
- Load project context via shell injections (keep this — it's good)
- Explicit invocation: `Skill(superpowers:brainstorming)` if not obvious fix, else `Skill(superpowers:test-driven-development)` directly
- Let Superpowers chain: brainstorming → writing-plans → subagent (for non-trivial) or TDD directly (for obvious fixes)
- Conditional doc update stays the same
- Add: invoke `Skill(gstack:careful)` for safety if touching production code
- No pipeline state file (keep this — it's lightweight)

### Skill 9: pipeline-qa (entry point)
**Changes:**
- Step 0: if no TESTING.md, run `Skill(pipeline-test-setup)` first
- Step 1: Read TESTING.md for project profile, parse test types and commands
- Step 2: Dispatch using Agent tool for actual parallelism (not sequential Skill calls):
  - Agent(gstack:review) for code review
  - Agent(gstack:cso) for security
  - Run test command via Bash for unit tests
  - Agent(gstack:qa) for browser QA (if web platform + browse available)
  - Run simulator command via Bash (if mobile platform)
  - Agent(gstack:benchmark) for performance (if web)
- Step 3: aggregate report with timestamps, tool versions, severity ranking
- Step 4: offer fixes via pipeline-quick
- Add: `Skill(superpowers:verification-before-completion)` before declaring QA complete
- Add: save report to `.gsd/qa-reports/YYYY-MM-DD.md`

## Templates to Update

### templates/CLAUDE.md
- Add `## Skill Routing` section (gstack expects this)
- Add `## Testing` section pointing to `.gsd/TESTING.md`
- Update Phase Delegation table with correct invocation syntax
- Add GSD workflow templates reference
- Remove `.gsd/` file descriptions we don't create (GSD manages those)

### templates/TESTING.md
- Keep as fallback/reference only
- Real TESTING.md is generated interactively by pipeline-test-setup
- Add clear header: "Generated by /pipeline-test-setup — edit as needed"

## Setup Script Updates

### setup
- Fix sed logic for gstack skill disabling (use node instead of brittle sed)
- Add verification step: check symlinks resolve correctly
- Add `gsd config` prompt if GSD has no config
- Add check for Node.js (required for settings.json update)
- Add `--reinstall` flag to force re-clone gstack

## File Changes Summary

| File | Action |
|------|--------|
| skills/pipeline-scaffold/SKILL.md | Rewrite — remove .gsd/ creation, delegate to gsd init |
| skills/pipeline-test-setup/SKILL.md | NEW — interactive test strategy questionnaire |
| skills/pipeline-doc-update/SKILL.md | Rewrite — add tool file awareness, explicit Edit usage |
| skills/pipeline-design-gate/SKILL.md | Rewrite — fix brand-kit ref, add AskUserQuestion, error handling |
| skills/pipeline-new/SKILL.md | Rewrite — explicit Skill() calls, proper phases, state tracking |
| skills/pipeline-init/SKILL.md | Rewrite — Agent(Explore), gsd init delegation, test setup |
| skills/pipeline-build/SKILL.md | Rewrite — GSD workflow templates, --type flag, proper state |
| skills/pipeline-quick/SKILL.md | Rewrite — explicit Skill() invocations, careful mode |
| skills/pipeline-qa/SKILL.md | Rewrite — TESTING.md-driven, Agent parallelism, verification |
| templates/CLAUDE.md | Update — skill routing, correct phase delegation |
| templates/TESTING.md | Update — mark as generated, reference format |
| CLAUDE.md | Update — reflect new skill count (9), test-setup mention |
| setup | Fix — sed logic, add verification, gsd config check |
