# [Project Name]

[One-line description]

## Tech Stack

- **Language:** [e.g., TypeScript]
- **Framework:** [e.g., Next.js 14]
- **Database:** [e.g., PostgreSQL via Prisma]
- **Hosting:** [e.g., Vercel]

## Quick Start

```bash
[install command, e.g., npm install]
[dev server command, e.g., npm run dev]
```

## Pipeline Commands

| Command | When | What happens |
|---------|------|--------------|
| `/pipeline-build` | New feature | Requirements → brainstorm → spec → plan → design → worktree → execute → finish branch → QA → docs → ship |
| `/pipeline-quick` | Small fix / bug | Routes to systematic-debugging (bugs), TDD (clear fixes), or brainstorming (features) |
| `/pipeline-qa` | Run QA | Reads TESTING.md → dispatches unit/review/security/browser/perf → reports → routes fixes |
| `/pipeline-init` | Re-init pipeline | Re-discover codebase, update docs, reconfigure testing |

### Flags for /pipeline-build

- `--spec <file>` — skip to planning (you have a spec)
- `--plan <file>` — skip to execution (you have a plan)
- `--resume` — continue interrupted pipeline (uses `gsd --continue` for GSD sessions)
- `--no-qa` — skip QA phase
- `--type bugfix|small|feature|refactor|hotfix|spike|dep-upgrade|security-audit` — execution strategy
- `--budget <N>` — token budget for GSD execution

## Skill Routing

- "brainstorm", "design a feature", "think through this" → `Skill(superpowers:brainstorming)`
- "debug this", "why is this broken" → `Skill(superpowers:systematic-debugging)` or `Skill(gstack:investigate)` for production
- "review this PR", "code review" → `Skill(gstack:review)`
- "security audit" → `Skill(gstack:cso)`
- "test this site", "QA" → `Skill(gstack:qa)` (fix loop) or `Skill(gstack:qa-only)` (report only)
- "check performance" → `Skill(gstack:benchmark)`
- "ship", "create PR" → `Skill(gstack:ship)` → `Skill(gstack:land-and-deploy)` → `Skill(gstack:canary)`
- "be careful", "production change" → `Skill(gstack:careful)`
- "what have we learned" → `Skill(gstack:learn)`

## Documentation

- `docs/ARCHITECTURE.md` — system design, data flow, where things live
- `docs/API.md` — endpoints, params, responses
- `docs/COMPONENTS.md` — UI component inventory
- `docs/MODELS.md` — data shapes, relationships
- `.gsd/TESTING.md` — test strategy (how this project is tested)
- `docs/superpowers/specs/` — design specs from brainstorming
- `docs/superpowers/plans/` — implementation plans

## Testing

Strategy defined in `.gsd/TESTING.md`. Quick reference:
- **Unit:** `[test command]`
- **QA:** `/pipeline-qa`

## Who Does What

**Superpowers** — brainstorming, spec writing, planning, TDD, systematic debugging, code review, subagent execution, worktree management, branch finishing, verification. The engine for all coding work. Skills auto-chain: brainstorming → writing-plans → worktree → subagent-driven-development → finishing-a-development-branch.

**GSD v2** (`gsd headless`) — parallel autonomous execution for large tasks (>10). Wave-based parallelism with atomic commits. Workers auto-load Superpowers. Templates: bugfix, small-feature, refactor, hotfix, spike, dep-upgrade, security-audit.

**gstack** — office-hours (requirements via 6 forcing questions), review (SQL safety, trust boundaries, conditional side effects), qa (browser test-fix loop), qa-only (browser report only), cso (security audit: daily/comprehensive), investigate (root cause debugging with browse daemon), browse (headless browser ~100ms), careful (destructive command guardrails), benchmark (Core Web Vitals, baselines, regression detection), learn (persistent learnings across sessions), ship (VERSION, CHANGELOG, review, PR), land-and-deploy (merge, CI, deploy verify), canary (post-deploy monitoring 30min).

**Paper** (MCP, optional) — UI component design exploration. Creates 3 design options for new components, user picks, exports JSX/style tokens to `.gsd/designs/`.

## Phase Delegation

| Phase | Owner | How |
|-------|-------|-----|
| Requirements | gstack `/office-hours` | 6 forcing questions, saves design doc |
| Brainstorm + Spec | Superpowers `brainstorming` | Interactive, outputs to `docs/superpowers/specs/` |
| Plan | Superpowers `writing-plans` | Auto-invoked by brainstorming, tasks with tags |
| Design gate | Pipeline + Paper MCP | 3 options per ui-new component |
| Worktree setup | Superpowers `using-git-worktrees` | Isolated branch, clean baseline |
| Execute (≤10 tasks) | Superpowers `subagent-driven-development` | Per-task dispatch, two-stage review |
| Execute (>10 tasks) | GSD `gsd headless auto` | Parallel waves, workers have Superpowers |
| Finish branch | Superpowers `finishing-a-development-branch` | 4 options: merge/PR/keep/discard |
| QA (report) | gstack `/qa-only` + `/cso` + `/benchmark` | Report only, no fixes |
| QA (fix bugs) | Superpowers `systematic-debugging` / `TDD` | Routed by issue type |
| QA (browser fix) | gstack `/qa` | Test-fix-verify loop |
| Doc update | Pipeline skill | Fork reads git diff, updates docs/ |
| Ship | gstack `/ship` | VERSION, CHANGELOG, review, PR |
| Deploy | gstack `/land-and-deploy` → `/canary` | Merge, CI, deploy verify, 30min monitor |
| Learnings | gstack `/learn` | Store patterns from this session |

## Rules

1. **Delegate, don't duplicate.** Use Superpowers skills and gstack commands. The pipeline only adds orchestration.
2. **Worktree before execution.** Always set up an isolated worktree via `using-git-worktrees` before running `subagent-driven-development` or `executing-plans`.
3. **Let Superpowers finish.** After execution, `finishing-a-development-branch` presents merge/PR/keep/discard options. Let it complete before offering gstack:ship.
4. **Report first, fix second.** QA uses `qa-only` (report). Fixes are routed through `systematic-debugging` (unclear bugs), `test-driven-development` (clear fixes), or `gstack:qa` (browser test-fix loops).
5. **No double review.** If `gstack:ship` will run, skip `gstack:review` in QA — ship runs review internally.
6. **Tag tasks in plans.** Every task must have one tag: `<!-- tag: ui-new -->`, `<!-- tag: ui-update -->`, or `<!-- tag: logic -->`.
7. **Design before implement.** `ui-new` tasks go through Paper design gate before execution.
8. **Docs stay current.** After any pipeline run, update docs/ to reflect what changed.
9. **Fallback gracefully.** If a tool isn't installed, use the next best option.
10. **Evidence before claims.** Always run `verification-before-completion` before declaring work done — fresh test output, not stale assumptions.
