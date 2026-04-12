---
name: pipeline-init
description: Set up the pipeline on a new or existing project. Deep-dives the codebase with parallel sub-agents, generates concise .gsd/ docs, configures testing, creates CLAUDE.md.
disable-model-invocation: true
user-invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Skill, Agent, AskUserQuestion
argument-hint: "[project name or description] (run in the project directory)"
---

# Pipeline Init

Set up pipeline infrastructure. Deep-dives the codebase to produce concise, high-signal docs.

Existing source files: !`ls src/ lib/ app/ *.py *.go *.rs main.* index.* 2>/dev/null | head -5 || echo "NONE"`
Existing CLAUDE.md: !`[ -f CLAUDE.md ] && head -3 CLAUDE.md || echo "NONE"`
Existing .gsd docs: !`ls .gsd/PROJECT.md .gsd/KNOWLEDGE.md .gsd/RUNTIME.md 2>/dev/null || echo "NONE"`
GSD installed: !`which gsd 2>/dev/null && echo "YES" || echo "NO"`
gstack installed: !`ls ~/.claude/skills/gstack/review/SKILL.md 2>/dev/null && echo "YES" || echo "NO"`

## Process

### 1. Detect Mode

- **Source files found** → existing project (skip Step 2)
- **No source files** → greenfield (run Step 2)

### 2. Greenfield Setup (skip if existing)

```bash
git init 2>/dev/null || true
mkdir -p src .gsd docs/superpowers/specs docs/superpowers/plans
echo "0.1.0" > VERSION
```

Ask user what they're building using `AskUserQuestion`:
- Header: "Project type"
- Options: "Web app (Next.js / React / Vue)", "API (Express / FastAPI / Django / Go)", "Mobile (React Native / Expo / Flutter)", "CLI / Library"

Create minimal directory structure. Don't scaffold code — just directories and a README stub.

### 3. Deep-Dive Codebase Discovery

This is the most important step. Dispatch **parallel sub-agents** to thoroughly analyze large codebases. Each agent focuses on one domain so nothing gets missed.

**Launch all 4 agents in parallel:**

```
Agent(subagent_type="Explore", prompt="ARCHITECTURE SCAN: Analyze the project structure deeply.
- Language, framework, runtime (check package.json, requirements.txt, go.mod, Cargo.toml, pyproject.toml, etc.)
- Every top-level and second-level directory — what lives in each
- Entry points: main files, route definitions, app bootstrap
- Build system, dev server commands, scripts in package.json
- Infrastructure: hosting, CI/CD configs, Dockerfiles, cloud configs
- Monorepo structure if applicable (workspaces, packages)
Return file paths for everything found. Be thorough — this is a large codebase.")
```

```
Agent(subagent_type="Explore", prompt="API & RUNTIME SCAN: Find every API endpoint and runtime dependency.
- All route files, HTTP handlers, middleware, GraphQL resolvers
- Map each endpoint: method, path, params, response shape
- Auth patterns (JWT, sessions, API keys, OAuth)
- Environment variables referenced in code (process.env, os.environ, etc.)
- External services: databases, caches, queues, third-party APIs
- WebSocket or real-time connections
Return file paths and specific line numbers for key definitions.")
```

```
Agent(subagent_type="Explore", prompt="DATA & UI SCAN: Find all data models and UI components.
- Database models/schemas (ORM models, migrations, SQL files)
- TypeScript interfaces/types that define domain objects
- Key relationships between models (foreign keys, references)
- UI components: inventory with file paths, shared vs page-specific
- Component hierarchy and key prop interfaces
- State management patterns (Redux, Zustand, Context, signals)
Return file paths for everything. Include line numbers for type definitions.")
```

```
Agent(subagent_type="Explore", prompt="PATTERNS & TESTING SCAN: Identify conventions and test infrastructure.
- Naming conventions: files (camelCase, kebab-case), variables, functions, classes
- Test framework, config files, test directories, existing test files
- Test patterns used (describe/it, test(), pytest fixtures, table-driven)
- Error handling patterns
- Logging approach
- Code organization conventions (barrel exports, index files, co-location)
- Git conventions (commit message style, branch naming)
Return specific examples of each pattern found, with file paths.")
```

Wait for all 4 agents to return. Merge their findings — this combined summary drives everything below.

### 4. Initialize GSD

If GSD is installed:
```bash
mkdir -p .gsd
gsd headless "init" --no-session
```

If GSD is NOT installed:
```bash
mkdir -p .gsd
```

### 5. Generate .gsd/ Docs

Write concise, high-signal docs. **Key facts only — file paths, relationships, behavior. No explanatory prose, no filler.**

**`.gsd/PROJECT.md`** — Living project doc:
```markdown
# Project

[One-line description]

## Architecture
- [Key dir] — [what it does]
- [Key dir] — [what it does]

## Data Flow
[request → middleware → handler → service → DB → response]

## Entry Points
- `path/to/main.ts` — app bootstrap
- `path/to/routes/` — route definitions

## Components
- `path/Component.tsx` — [purpose]

## Models
- `path/Model.ts` — [fields, relationships]

## Infrastructure
- [hosting, DB, external services]
```

**`.gsd/RUNTIME.md`** — Runtime context:
```markdown
# Runtime

## Endpoints
- `GET /api/users` — list users, paginated
- `POST /api/users` — create user, requires auth

## Environment Variables
- `DATABASE_URL` — PostgreSQL connection
- `JWT_SECRET` — auth token signing

## Services
- PostgreSQL (primary DB)
- Redis (session cache)
```

**`.gsd/KNOWLEDGE.md`** — Patterns and conventions:
```markdown
# Knowledge

## Naming
- Files: kebab-case
- Components: PascalCase
- Variables: camelCase

## Testing
- Framework: vitest
- Command: `npm test`
- Pattern: describe/it with factory helpers

## Patterns
- [Key pattern with file path example]
```

**Format rules:**
- Bullet points and tables, not paragraphs
- Every claim has a file path
- If grep can find it in 5 seconds, don't document it
- Maximum 80 lines per doc for a medium project — scale proportionally

**If files already exist** (from a previous GSD milestone), read first, enrich with Edit tool. Never overwrite GSD content.

```bash
mkdir -p docs/superpowers/specs docs/superpowers/plans
```

### 6. Generate CLAUDE.md

**If CLAUDE.md exists:** Read it. Use Edit to add missing sections (Documentation table, Testing, Grounding Rules, Tools Reference, Workflow Guidance, Pipeline Commands).

**If no CLAUDE.md:** Populate the template with discovery results:
- Project name, description, tech stack, quick start commands
- Test commands from discovery
- Conventions from discovery
- Keep grounding rules, tools reference, workflow guidance as-is

Present to user via `AskUserQuestion`:
- Header: "CLAUDE.md"
- Options: "Looks good", "I want to edit it", "Regenerate with changes"

### 7. Configure Testing

```
Skill(pipeline-test-setup)
```

### 8. Seed Learnings

If gstack learn is available:
```
Skill(gstack:learn)
```

### 9. Commit

```bash
git add .gsd/ docs/ CLAUDE.md .gitignore VERSION 2>/dev/null
git diff --cached --quiet || git commit -m "chore: initialize pipeline infrastructure"
```

## Done

Report:
- Key findings from discovery
- `.gsd/` docs created (PROJECT.md, RUNTIME.md, KNOWLEDGE.md)
- Test strategy configured
- Commands: `/pipeline-build`, `/pipeline-qa`, `/pipeline-doc-update`, `/pipeline-test-setup`
