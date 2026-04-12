---
name: pipeline-init
description: Add the pipeline system to an existing project. Discovers codebase, generates docs, configures testing, creates CLAUDE.md.
disable-model-invocation: true
user-invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Skill, Agent, AskUserQuestion
argument-hint: (run in an existing project directory)
---

# Pipeline Init

Add pipeline infrastructure to an existing project without changing any code.

Existing CLAUDE.md: !`[ -f CLAUDE.md ] && head -5 CLAUDE.md || echo "NONE"`
Existing docs: !`ls docs/*.md 2>/dev/null || echo "NONE"`
Existing .gsd: !`ls .gsd/ 2>/dev/null || echo "NONE"`
GSD installed: !`which gsd 2>/dev/null && echo "YES" || echo "NO"`
gstack installed: !`ls ~/.claude/skills/gstack/review/SKILL.md 2>/dev/null && echo "YES" || echo "NO"`

## Process

### 1. Discover Codebase

Dispatch an Explore agent to scan the project:

```
Agent(subagent_type="Explore", prompt="Analyze this project thoroughly:
- Language and framework (check package.json, requirements.txt, go.mod, Cargo.toml, etc.)
- Directory structure and entry points
- Dependencies and their purposes
- Test setup: framework, config files, existing test files, test scripts
- Build system and dev server commands
- Database / API layer
- Existing documentation
- CI/CD configuration
- Key architecture patterns (monolith, microservices, serverless, etc.)
- Naming conventions visible in the code
Return a structured summary with exact file paths for everything found.")
```

Wait for the agent to return. This summary drives everything below.

### 2. Create directory structure

```bash
mkdir -p docs .gsd/designs .claude
```

Only create what doesn't exist. **Never overwrite** existing files.

### 3. Initialize GSD

If GSD is installed:
```bash
gsd headless "init" --no-session
```

GSD creates `.gsd/CODEBASE.md` (project file map) and `.gsd/STATE.md`.

If GSD is NOT installed, skip this step (pipeline creates what it needs in Step 3a).

### 3a. Create pipeline-owned files

These files are pipeline-managed, not GSD-managed. Always create them:

- `.gsd/KNOWLEDGE.md` — Populated with patterns discovered in Step 1:
  - Naming conventions (camelCase? snake_case?)
  - Test patterns (describe/it? test()? pytest?)
  - Architecture decisions visible in code
  - Key file paths a subagent would need
- `.gsd/REQUIREMENTS.md` — `# Requirements` heading only

### 4. Generate docs

Based on discovery results, use the Write tool to create each doc. **Only include information discovered from actual code — never guess.**

**`docs/ARCHITECTURE.md`** — System design from discovery:
- Key directories and what lives in each
- Data flow (e.g., "request → middleware → handler → DB → response")
- Infrastructure (hosting, database, external services)
- Entry points (main file, route definitions, etc.)
- File paths for everything referenced

**`docs/API.md`** — If API endpoints discovered:
- Routes with HTTP methods
- Key params and response shapes
- Auth requirements
- If no API found, write: "# API\n\nNo API layer detected. Add endpoints here as they're built."

**`docs/COMPONENTS.md`** — If UI components discovered:
- Component inventory with file paths
- Key component hierarchies
- Shared/reusable components vs page-specific
- If no UI found, write: "# Components\n\nNo UI components detected. Add components here as they're built."

**`docs/MODELS.md`** — If data models discovered:
- Types/interfaces with file paths
- Database schema (if migrations found)
- Key relationships
- If no models found, write: "# Models\n\nNo data models detected. Add models here as they're built."

**If any doc file already exists**, read it first and only add missing information using the Edit tool.

### 5. Generate CLAUDE.md

**If CLAUDE.md exists:** Read it. Use Edit to add these sections if missing:
- `## Pipeline Commands` table
- `## Skill Routing` section
- `## Documentation` links
- `## Testing` reference to `.gsd/TESTING.md`

**If no CLAUDE.md:** Use Write to create one:

```markdown
# [Project Name]

[Description from discovery]

## Tech Stack

[From discovery — language, framework, database, hosting]

## Quick Start

[From discovery — install command, dev server command]

## Pipeline Commands

| Command | When | What happens |
|---------|------|--------------|
| `/pipeline-build` | New feature | Requirements → brainstorm → spec → plan → design → execute → QA → docs |
| `/pipeline-quick` | Small fix | Superpowers handles it, conditional doc update |
| `/pipeline-qa` | Run QA | Code review + security + browser testing + unit tests |

## Skill Routing

- "brainstorm", "design", "plan a feature" → `Skill(superpowers:brainstorming)`
- "review this PR", "code review" → `Skill(gstack:review)`
- "security audit" → `Skill(gstack:cso)`
- "test this site", "QA" → `Skill(gstack:qa)`
- "debug this" → `Skill(gstack:investigate)`
- "ship", "create PR" → `Skill(gstack:ship)`

## Documentation

- `docs/ARCHITECTURE.md` — system design, data flow
- `docs/API.md` — endpoints, params, responses
- `docs/COMPONENTS.md` — UI component inventory
- `docs/MODELS.md` — data shapes, relationships
- `.gsd/TESTING.md` — test strategy

## Testing

See `.gsd/TESTING.md` for full test strategy.

## Conventions

[From discovery — naming, testing patterns, commit style]
```

Present the CLAUDE.md to the user. Ask if they want changes using `AskUserQuestion`.

### 6. Configure Testing

Invoke: `Skill(pipeline-test-setup)`

This runs the interactive test strategy questionnaire and writes `.gsd/TESTING.md`. The questionnaire auto-detects what it can from the existing project before asking questions.

### 7. Seed Learnings

If gstack learn is available:
```
Skill(gstack:learn)
```

This reviews and stores the patterns discovered during init for future sessions.

### 8. Update .gitignore

Append if not present:
```
.claude/pipeline-state.json
```

### 9. Commit

```bash
git add docs/ .gsd/ CLAUDE.md .gitignore VERSION 2>/dev/null
git diff --cached --quiet || git commit -m "chore: initialize pipeline infrastructure"
```

## Done

Report:
- What was discovered about the project
- What docs were created/updated
- What test strategy was configured
- The project is now ready for `/pipeline-build`, `/pipeline-quick`, and `/pipeline-qa`
