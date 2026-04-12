# [Project Name]

[One-line description]

## Tech Stack

- **Language:** [e.g., TypeScript]
- **Framework:** [e.g., Next.js 14]
- **Database:** [e.g., PostgreSQL via Prisma]
- **Hosting:** [e.g., Vercel]

## Quick Start

```bash
[install command]
[dev server command]
```

## Documentation

All canonical project docs live in `.gsd/`. This is the single source of truth — all tools read from here.

| File | What it covers | Updated by |
|------|---------------|-----------|
| `.gsd/PROJECT.md` | Architecture, system design, data flow, components, models | GSD (during milestones) or `pipeline-doc-update` (after Superpowers work) |
| `.gsd/RUNTIME.md` | API endpoints, env vars, services | GSD or `pipeline-doc-update` |
| `.gsd/KNOWLEDGE.md` | Conventions, patterns, cross-session lessons | GSD or `pipeline-doc-update` |
| `.gsd/DECISIONS.md` | Architectural decisions (append-only log) | GSD or `pipeline-doc-update` (append only) |
| `.gsd/TESTING.md` | Test strategy — how this project is tested | `/pipeline-test-setup` only |
| `.gsd/CODEBASE.md` | Project file map | `gsd init` (auto-generated) |
| `docs/superpowers/specs/` | Decision records from brainstorming | Superpowers (ephemeral) |
| `docs/superpowers/plans/` | Implementation plans | Superpowers (ephemeral) |

**Doc update rules:**
- After **GSD execution**: docs are already current (GSD updates them during milestones)
- After **Superpowers execution**: run `/pipeline-doc-update` — it reads git diff and updates `.gsd/` files
- `pipeline-doc-update` also runs `gsd headless "init"` to refresh CODEBASE.md (the file map)
- Never manually edit `.gsd/STATE.md` — GSD's internal state machine

## Testing

Strategy defined in `.gsd/TESTING.md`. Quick reference:
- **Unit:** `[test command]`
- **QA:** `/pipeline-qa`

## Grounding Rules

1. **Read before acting.** Start every task by reading this file + `.gsd/PROJECT.md` + `.gsd/KNOWLEDGE.md`. Never assume — verify.
2. **Follow TESTING.md.** Run tests using the exact commands in `.gsd/TESTING.md`. Don't improvise test commands.
3. **Evidence before claims.** Show fresh test output before declaring work done. No stale assumptions.
4. **Update docs after structural changes.** Run `/pipeline-doc-update` after adding routes, models, components, or changing architecture.
5. **Atomic commits.** One logical change per commit. Conventional messages: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`.

## Tools Reference

### Superpowers (thinking + coding)

| Skill | When to use |
|-------|------------|
| `brainstorming` | Exploring a new feature, design decisions, architecture choices |
| `writing-plans` | Auto-invoked by brainstorming — creates task breakdown |
| `test-driven-development` | Writing a feature with clear requirements — test first, then implement |
| `systematic-debugging` | Bug with unclear cause — 4-phase root cause investigation |
| `subagent-driven-development` | Executing a plan with multiple tasks (auto uses worktree) |
| `finishing-a-development-branch` | After execution — merge, PR, keep, or discard the branch |
| `verification-before-completion` | Final check — runs tests with fresh output |

### GSD v2 (parallel autonomous execution)

| Command | When to use |
|---------|------------|
| `gsd headless "init"` | Initialize GSD state (creates CODEBASE.md + STATE.md) |
| `gsd headless new-milestone --context <spec> --auto` | Large tasks — takes a Superpowers spec as input, parallel execution |
| `gsd headless "start <template> <desc>"` | Start a workflow from a template |
| Templates: `bugfix`, `small-feature`, `refactor`, `hotfix`, `spike`, `dep-upgrade`, `security-audit` | Pick based on task type |

GSD creates its own branch and manages docs (PROJECT.md, KNOWLEDGE.md, RUNTIME.md, DECISIONS.md) during milestones. Skip worktree setup when using GSD.

### gstack (operations)

| Skill | When to use |
|-------|------------|
| `/office-hours` | Thinking through a new idea — 6 forcing questions |
| `/review` | Pre-landing code review (SQL safety, trust boundaries) |
| `/qa` | Browser test-fix-verify loop |
| `/qa-only` | Browser QA report only — no fixes |
| `/cso` | Security audit (OWASP, STRIDE, dependency scanning) |
| `/investigate` | Production debugging with browse daemon |
| `/benchmark` | Performance baseline + regression detection |
| `/ship` | VERSION bump, CHANGELOG, review, PR creation |
| `/land-and-deploy` | Merge, CI, deploy, verify |
| `/canary` | Post-deploy monitoring (30min) |
| `/learn` | Store patterns from this session |
| `/careful` | Safety guardrails for destructive commands |

## Workflow Guidance

### Building a new feature

Use `/pipeline-build <description>` for the full orchestrated flow, or run steps manually:

1. Read `.gsd/PROJECT.md` + `.gsd/KNOWLEDGE.md` for context
2. `Skill(superpowers:brainstorming)` — explore approaches, output spec
3. Plan auto-generates from brainstorming
4. Choose execution: Superpowers (focused, interactive) or GSD (large tasks, autonomous)
5. After execution: `/pipeline-doc-update` to sync `.gsd/` docs
6. `/pipeline-qa` — tests + review + security. Fixes and re-runs until clean.
7. `Skill(gstack:ship)` — when ready to land

### Fixing a bug

- **Unclear cause:** `Skill(superpowers:systematic-debugging)` — 4-phase investigation
- **Clear fix needed:** `Skill(superpowers:test-driven-development)` — write failing test, then fix
- **Production error:** `Skill(gstack:investigate)` — has browse daemon + learnings

### Quick small change

1. Make the change directly
2. Run tests from `.gsd/TESTING.md`
3. If structure changed, run `/pipeline-doc-update`

## Pipeline Commands

| Command | Purpose |
|---------|---------|
| `/pipeline-build` | Build a feature — brainstorm → plan → execute (Superpowers or GSD) → doc update → QA → iterate → ship |
| `/pipeline-init` | Set up pipeline on a new or existing project (discovery, docs, testing, CLAUDE.md) |
| `/pipeline-qa` | Run QA — reads TESTING.md, dispatches tests, produces severity-ranked report |
| `/pipeline-doc-update` | Update `.gsd/` docs from recent code changes (reads git diff) |
| `/pipeline-test-setup` | Interactive test strategy questionnaire — writes .gsd/TESTING.md |

## Conventions

[From discovery — naming patterns, file organization, commit style, etc.]
