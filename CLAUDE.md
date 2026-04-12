# coolcode

Pipeline framework for Claude Code that orchestrates Superpowers, GSD v2, and gstack into a unified development workflow.

## Repo Structure

```
coolcode/
├── setup                  ← install script (run this first)
├── CLAUDE.md              ← you're reading this
├── skills/                ← pipeline skill source of truth
│   ├── pipeline-new/      ← /pipeline-new — full project from scratch
│   ├── pipeline-init/     ← /pipeline-init — add pipeline to existing project
│   ├── pipeline-build/    ← /pipeline-build — feature with full pipeline
│   ├── pipeline-quick/    ← /pipeline-quick — small fix
│   ├── pipeline-qa/       ← /pipeline-qa — standalone QA
│   ├── pipeline-scaffold/ ← helper: project scaffolding
│   ├── pipeline-test-setup/ ← helper: interactive test strategy questionnaire
│   ├── pipeline-doc-update/ ← helper: living doc updates
│   └── pipeline-design-gate/ ← helper: Paper design exploration
├── templates/
│   ├── CLAUDE.md          ← project-level CLAUDE.md template
│   └── TESTING.md         ← per-project test strategy template
└── gstack-config/
    └── enabled-skills.txt ← which gstack skills stay active
```

## Editing Skills

All skills in `skills/` are symlinked to `~/.claude/skills/` by the setup script. Edit them here — changes take effect on next Claude Code session.

## Dependencies

| Tool | Purpose | Install |
|------|---------|---------|
| Superpowers | Brainstorming, planning, TDD, code review, subagent execution | Claude Code plugin marketplace |
| GSD v2 | Parallel autonomous execution for large tasks | `npm install -g gsd-pi@latest` |
| gstack | QA, security, browser testing, deployment | Cloned by setup script |
| Paper | UI component design exploration (optional) | MCP server |

## gstack Skill Configuration

`gstack-config/enabled-skills.txt` lists which gstack skills are active. All others get `disable-model-invocation: true`. Re-run `./setup` after editing the list.

Currently enabled: review, qa, cso, office-hours, investigate, browse, careful, qa-only, benchmark, learn, ship, land-and-deploy, canary.

## For Users

1. Clone this repo
2. Run `./setup`
3. Restart Claude Code
4. Copy `templates/CLAUDE.md` into your project root and customize
5. Use `/pipeline-init` to set up an existing project, or `/pipeline-new` to start fresh
