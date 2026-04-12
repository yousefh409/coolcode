---
name: pipeline-qa
description: Run QA on the current codebase. Reads .gsd/TESTING.md to know what to run. Supports --security, --browser, --review, --perf, --unit, --diff flags.
disable-model-invocation: true
user-invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Skill, Agent, AskUserQuestion
argument-hint: "[--security|--browser|--review|--perf|--unit|--simulator|--diff branch|--all]"
---

# Pipeline QA

Standalone QA — run anytime on the current codebase state. Reads `.gsd/TESTING.md` for test strategy.

**Input:** `$ARGUMENTS` = optional flags

gstack installed: !`ls ~/.claude/skills/gstack/review/SKILL.md 2>/dev/null && echo "YES" || echo "NO"`
Test strategy: !`[ -f .gsd/TESTING.md ] && head -20 .gsd/TESTING.md || echo "NO TESTING.md"`

## Flag Parsing

| Flag | Effect |
|------|--------|
| `--security` | Security audit only |
| `--browser` | Browser QA only |
| `--review` | Code review only |
| `--perf` | Performance benchmark only |
| `--simulator` | Simulator-based testing only |
| `--unit` | Run project test runner only |
| `--diff <branch>` | Only review changes since branch divergence |
| `--all` | Run everything regardless of TESTING.md |
| `--skip-review` | Skip code review (use when `gstack:ship` will run afterwards — ship reviews internally) |
| No flags | Run all applicable checks from `.gsd/TESTING.md` |

Multiple flags can be combined: `--review --security --unit`

## Process

### 0. Load Test Strategy

If `.gsd/TESTING.md` does not exist or is empty:
```
Skill(pipeline-test-setup)
```
Wait for test setup to complete before continuing.

Read `.gsd/TESTING.md` to understand:
- **Platform** — web, mobile, desktop, api, cli, library
- **Test runner** — jest, vitest, pytest, etc. and the exact command
- **Test types configured** — which sections exist in TESTING.md
- **Dev server / deployed URL** — where to point browser tests
- **Environment setup** — env vars, setup commands needed before tests

### 1. Determine Scope

If `--diff <branch>` provided:
```bash
git diff --name-only <branch>...HEAD
```
Use changed files to narrow which test types are most relevant. Still run requested types, but note scope.

If specific flags provided, only run those types.

If no flags, run all test types that have sections in TESTING.md.

### 2. Dispatch Tests

Run applicable checks. Use the `Agent` tool to parallelize independent checks.

**Important ordering:**
1. **First:** Run unit/integration tests (fast, catches regressions early)
2. **Then:** Dispatch review, security, browser, simulator, perf in parallel via Agent tool
3. **Last:** Run `Skill(superpowers:verification-before-completion)` as final sanity check

#### Unit / Integration Tests

If TESTING.md has a test command configured:

```bash
# Run exact command from TESTING.md
<test-command>
```

Run setup commands first if TESTING.md specifies them (env vars, seed DB, start services).

Report: pass/fail counts, failing test names, relevant error output.

#### Code Review

**Skip if `--skip-review` flag is set.** Use this flag when `gstack:ship` will run after QA — ship runs `gstack:review` internally and you'd be reviewing the same diff twice.

If gstack installed:
```
Skill(gstack:review)
```

If gstack NOT installed, dispatch an agent:
```
Agent(subagent_type="gsd-code-reviewer", prompt="Review the codebase for bugs, security issues, and code quality. Focus on: [changed files if --diff, otherwise recent changes]")
```

#### Security Audit

If gstack installed:
```
Skill(gstack:cso)
```
This runs in daily mode (8/10 confidence gate) by default. For comprehensive monthly scan, add `--comprehensive` context.

If gstack NOT installed, dispatch an agent:
```
Agent(subagent_type="security-auditor", prompt="Scan the codebase for security vulnerabilities: OWASP Top 10, hardcoded secrets, injection risks, auth issues, insecure dependencies.")
```

#### Browser QA

Only run if platform is web (from TESTING.md profile).

**Use `gstack:qa-only` — report only, no fixes:**
```
Skill(gstack:qa-only)
```
Point at the dev server or deployed URL from TESTING.md.

This produces a health score, annotated screenshots, and repro steps for each bug. It does NOT modify code or make commits.

If gstack NOT installed but Playwright/Cypress configured in TESTING.md:
```bash
<e2e-command-from-TESTING.md>
```

#### Simulator QA

Only run if platform is mobile (from TESTING.md profile).

Run the simulator test command from TESTING.md:
```bash
<simulator-command>
```

If no automation configured, report: "No simulator automation — manual testing required."

#### Performance

If gstack installed and platform is web:
```
Skill(gstack:benchmark)
```
First run establishes baselines. Subsequent runs compare against baselines and alert on regressions (>5% change).

If custom perf command in TESTING.md:
```bash
<perf-command>
```

#### Visual Regression

If `.gsd/designs/` has exports:
- Read design specs from `.gsd/designs/task-*.md`
- Compare implemented UI against approved design descriptions
- Flag significant drift in layout, colors, typography

### 3. Verification

After all test types complete, run final verification:
```
Skill(superpowers:verification-before-completion)
```
This runs the test command (from TESTING.md, CLAUDE.md, or package.json) with fresh output and refuses to claim success without evidence.

### 4. Aggregate Report

Combine all findings into a severity-ranked report:

```markdown
# QA Report — YYYY-MM-DD

## Summary

- **Scope:** [all / diff from branch / specific flags]
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
- [test type]: no issues found
```

Save to `.gsd/qa-reports/qa-report-YYYY-MM-DD.md`.

Create the directory if needed:
```bash
mkdir -p .gsd/qa-reports
```

### 5. Fix Routing

If critical or important issues were found:

Use `AskUserQuestion`:
- "Found N critical and M important issues. How should I fix them?" (Header: "Fixes")
- Options:
  - "Fix automatically" — routes each issue to the right tool (see below)
  - "Fix critical only" — same routing, critical issues only
  - "Don't fix — report only" — stop here

**Fix routing by issue type:**

| Issue Type | Tool | Why |
|-----------|------|-----|
| Bug with clear root cause | `Skill(superpowers:test-driven-development)` | Write failing test, then fix |
| Bug with unclear cause | `Skill(superpowers:systematic-debugging)` | 4-phase root cause investigation before any fix |
| Production error / stack trace | `Skill(gstack:investigate)` | Has browse daemon access, learnings framework |
| Browser UI bug | `Skill(gstack:qa)` | Full test-fix-verify loop with headless browser |
| Security vulnerability | `Skill(superpowers:test-driven-development)` | Write test proving vuln, then fix |
| Performance regression | `Skill(gstack:benchmark)` after fix | Fix manually, then re-benchmark |

For each fix:
1. Apply the fix using the routed tool
2. Re-run the specific test that caught the issue to verify
3. Commit the fix atomically

After all fixes, re-run the full QA suite to verify no regressions.

## Done

Report summary: X critical, Y important, Z minor issues found. Link to the full report at `.gsd/qa-reports/`.
