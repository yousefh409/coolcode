---
name: pipeline-qa
description: Run QA on the current codebase. Reads .gsd/TESTING.md for test strategy. Supports --unit, --review, --security, --browser, --fix, --all flags.
disable-model-invocation: true
user-invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Skill, Agent, AskUserQuestion
argument-hint: "[--unit|--review|--security|--browser|--fix|--all]"
---

# Pipeline QA

Run QA on the current codebase. Reads `.gsd/TESTING.md` for test strategy, dispatches checks, produces a severity-ranked report.

**Input:** `$ARGUMENTS` = optional flags

gstack installed: !`ls ~/.claude/skills/gstack/review/SKILL.md 2>/dev/null && echo "YES" || echo "NO"`
Test strategy: !`[ -f .gsd/TESTING.md ] && head -20 .gsd/TESTING.md || echo "NO TESTING.md"`

## Flags

| Flag | Effect |
|------|--------|
| `--unit` | Unit/integration tests only |
| `--review` | Code review only |
| `--security` | Security audit only |
| `--browser` | Browser QA only |
| `--fix` | Auto-fix issues and re-run until clean (used by /pipeline-build) |
| `--all` | Run everything regardless of TESTING.md |
| No flags | Run all applicable checks from TESTING.md |

Multiple flags can be combined: `--review --security --unit --fix`

## Process

### 1. Load Test Strategy

If `.gsd/TESTING.md` does not exist:
```
Skill(pipeline-test-setup)
```
Wait for test setup to complete before continuing.

Read `.gsd/TESTING.md` to understand:
- **Platform** — web, mobile, api, cli, library
- **Test runner** and exact command
- **Test types configured** — which sections exist
- **Dev server / URL** — where to point browser tests

### 2. Determine Scope

If specific flags provided, only run those types.
If no flags, run all test types that have sections in TESTING.md.

### 3. Dispatch Tests

**Order matters:**
1. **First:** Unit/integration tests (fast, catches regressions early)
2. **Then:** Dispatch review, security, browser in parallel via Agent tool

#### Unit / Integration Tests

Run the exact command from TESTING.md. Run setup commands first if specified (env vars, seed DB, start services).

Report: pass/fail counts, failing test names, relevant error output.

#### Code Review

If gstack installed:
```
Skill(gstack:review)
```

If gstack NOT installed:
```
Agent(subagent_type="gsd-code-reviewer", prompt="Review the codebase for bugs, security issues, and code quality.")
```

#### Security Audit

If gstack installed:
```
Skill(gstack:cso)
```

If gstack NOT installed:
```
Agent(subagent_type="security-auditor", prompt="Scan for OWASP Top 10, hardcoded secrets, injection risks, auth issues, insecure dependencies.")
```

#### Browser QA

Only if platform is web (from TESTING.md).

```
Skill(gstack:qa-only)
```

Point at the dev server or deployed URL from TESTING.md. This produces a health score, screenshots, and repro steps — no fixes.

If gstack NOT installed but E2E framework configured:
```bash
<e2e-command-from-TESTING.md>
```

### 4. Aggregate Report

Combine all findings into a severity-ranked report:

```markdown
# QA Report — YYYY-MM-DD

## Summary
- **Tests run:** [list of test types executed]
- **Overall:** X critical, Y important, Z minor

## Critical (must fix before shipping)
- [finding with file path and description]

## Important (should fix)
- [finding with file path and description]

## Minor (nice to fix)
- [finding with file path and description]

## Passed
- [test type]: all passing (N tests)
```

Save to `.gsd/qa-reports/qa-report-YYYY-MM-DD.md`:
```bash
mkdir -p .gsd/qa-reports
```

### 5. Fix + Iterate (when `--fix` flag is set)

If `--fix` is NOT set: report summary and suggest tools for fixing. Stop here.

If `--fix` IS set (used by `/pipeline-build`):

1. **Present ALL findings** — show the complete report with every issue, severity, file path
2. **Ask qualifying questions** using `AskUserQuestion`:
   - Header: "Fix approach"
   - Question: "QA found N critical and M important issues. How should I approach fixes?"
   - Options:
     - "Fix all" — description: "Fix every critical and important issue, then re-run QA"
     - "Fix critical only" — description: "Fix only critical issues, leave important as known"
     - "Let me review first" — description: "I'll look at the report and tell you what to fix"
3. **Auto-fix** each issue using the appropriate tool:
   - Bug with unclear cause → `Skill(superpowers:systematic-debugging)`
   - Bug with clear fix → `Skill(superpowers:test-driven-development)`
   - Browser UI bug → `Skill(gstack:qa)`
   - Security vulnerability → `Skill(superpowers:test-driven-development)` (write test proving vuln, then fix)
4. **Re-run QA** — go back to Step 3 (dispatch tests) with the same scope
5. **Repeat** until no critical/important issues remain

## Done

Report summary: X critical, Y important, Z minor issues found. Link to the full report at `.gsd/qa-reports/`.

If `--fix` was used, also report: number of fix iterations, what was fixed, final QA status.
