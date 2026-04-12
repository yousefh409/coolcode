# coolcode

Pipeline framework for Claude Code that orchestrates Superpowers, GSD v2, and gstack into a unified development workflow.

## Repo Structure

```
coolcode/
├── setup                    ← install script (run this first)
├── CLAUDE.md                ← you're reading this
├── skills/                  ← pipeline skill source of truth
│   ├── pipeline-build/      ← /pipeline-build — end-to-end feature build
│   ├── pipeline-init/       ← /pipeline-init — set up pipeline on any project
│   ├── pipeline-qa/         ← /pipeline-qa — standalone QA
│   ├── pipeline-doc-update/ ← /pipeline-doc-update — update .gsd/ docs from code changes
│   └── pipeline-test-setup/ ← /pipeline-test-setup — interactive test strategy
├── templates/
│   ├── CLAUDE.md            ← project-level CLAUDE.md template (the orchestration layer)
│   └── TESTING.md           ← per-project test strategy template
└── gstack-config/
    └── enabled-skills.txt   ← which gstack skills stay active
```

## Editing Skills

All skills in `skills/` are symlinked to `~/.claude/skills/` by the setup script. Edit them here — changes take effect on next Claude Code session.

## Dependencies

| Tool | Purpose | Install |
|------|---------|---------|
| Superpowers | Brainstorming, planning, TDD, code review, subagent execution | Claude Code plugin marketplace |
| GSD v2 | Parallel autonomous execution for large tasks | `npm install -g gsd-pi@latest` |
| gstack | QA, security, browser testing, deployment | Cloned by setup script |

## Documentation Architecture

`.gsd/` is the canonical doc home for every project. All tools read from `.gsd/PROJECT.md`, `.gsd/KNOWLEDGE.md`, `.gsd/RUNTIME.md`, `.gsd/DECISIONS.md`. GSD updates these during milestones; `pipeline-doc-update` updates them after Superpowers execution (reads git diff). Superpowers specs/plans in `docs/superpowers/` are ephemeral decision records.

## gstack Skill Configuration

`gstack-config/enabled-skills.txt` lists which gstack skills are active. All others get `disable-model-invocation: true`. Re-run `./setup` after editing the list.

Currently enabled: review, qa, cso, office-hours, investigate, browse, careful, qa-only, benchmark, learn, ship, land-and-deploy, canary.

## For Users

1. Clone this repo
2. Run `./setup`
3. Restart Claude Code
4. Run `/pipeline-init` in your project directory
