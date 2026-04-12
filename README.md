# coolcode

Pipeline framework for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that orchestrates **Superpowers**, **GSD v2**, and **gstack** into a unified development workflow.

One install. Five commands. Full lifecycle coverage — from brainstorming to shipping.

## Quick Start

```bash
git clone https://github.com/yousefh409/coolcode.git
cd coolcode
./setup
```

Restart Claude Code, then run `/pipeline-init` in any project directory.

## Commands

| Command | What it does |
|---------|-------------|
| `/pipeline-init` | Set up the pipeline on a new or existing project. Deep-dives the codebase with parallel sub-agents, writes concise `.gsd/` docs. |
| `/pipeline-build` | End-to-end feature build: brainstorm → plan → execute (Superpowers or GSD) → update docs → QA → iterate until clean → ship. |
| `/pipeline-qa` | Run QA against your `.gsd/TESTING.md` strategy. Use `--fix` for auto-iterate mode. |
| `/pipeline-test-setup` | Interactive questionnaire to define how your project is tested. Writes `.gsd/TESTING.md`. |
| `/pipeline-doc-update` | Update `.gsd/` docs from recent code changes. Reads git diff, updates concisely. |

## How It Works

```
You describe a feature
        │
        ▼
  /pipeline-build
        │
        ├─ Brainstorm (Superpowers)
        │      ↓
        ├─ Plan (Superpowers)
        │      ↓
        ├─ Execute ──┬── Superpowers (interactive, worktree-isolated)
        │            └── GSD v2 (autonomous, parallel waves)
        │      ↓
        ├─ Update docs (.gsd/)
        │      ↓
        ├─ QA + iterate (test, fix, re-test)
        │      ↓
        └─ Ship (version bump, changelog, PR)
```

## Dependencies

| Tool | Purpose | Install |
|------|---------|---------|
| [Superpowers](https://github.com/superpowers-ai/superpowers) | Brainstorming, planning, TDD, code review, subagent execution | Claude Code plugin marketplace |
| [GSD v2](https://github.com/gsd-build/gsd-2) | Parallel autonomous execution for large tasks | `npm install -g gsd-pi@latest` |
| [gstack](https://github.com/gstack-build/gstack) | QA, security, browser testing, deployment | Cloned by `./setup` |

All three are installed/verified by `./setup`. Superpowers must be installed manually from the Claude Code plugin marketplace.

## Documentation Architecture

Every project using coolcode keeps canonical docs in `.gsd/`:

| File | Purpose | Updated by |
|------|---------|-----------|
| `PROJECT.md` | Architecture, components, data models | GSD (milestones) + `pipeline-doc-update` |
| `KNOWLEDGE.md` | Conventions, patterns, lessons learned | GSD (milestones) + `pipeline-doc-update` |
| `RUNTIME.md` | API endpoints, env vars, services | GSD (milestones) + `pipeline-doc-update` |
| `DECISIONS.md` | Architectural decisions (append-only) | GSD (milestones) + `pipeline-doc-update` |
| `TESTING.md` | Test strategy and commands | `pipeline-test-setup` |
| `CODEBASE.md` | File map | `gsd init` (auto-generated) |
| `STATE.md` | GSD state machine | GSD only (never edit) |

## Repo Structure

```
coolcode/
├── setup                    ← install script (run this first)
├── CLAUDE.md                ← project instructions for Claude Code
├── skills/                  ← pipeline skill definitions
│   ├── pipeline-build/
│   ├── pipeline-init/
│   ├── pipeline-qa/
│   ├── pipeline-doc-update/
│   └── pipeline-test-setup/
├── templates/
│   ├── CLAUDE.md            ← project-level CLAUDE.md template
│   └── TESTING.md           ← test strategy template reference
└── gstack-config/
    └── enabled-skills.txt   ← which gstack skills stay active
```

## Editing Skills

All skills in `skills/` are symlinked to `~/.claude/skills/` by the setup script. Edit them here — changes take effect on next Claude Code session.

## License

MIT
