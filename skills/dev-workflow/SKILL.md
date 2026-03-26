---
name: dev-workflow
description: Guided development workflow with automated plan review, implementation, testing, and code review. Orchestrates plan → review → implement → check/test → code review → rules update.
allowed-tools: Read, Write, Edit, Glob, Grep, TodoWrite, EnterPlanMode, ExitPlanMode, Skill(ask-peer), Skill(ask-claude), Skill(ask-codex), Skill(ask-gemini), Skill(ask-copilot), Skill(extract-rules), Skill(simplify), Bash(pwd), Bash(pnpm run *), Bash(pnpm exec *), Bash(npm run *), Bash(yarn run *), Bash(bun run *), Bash(bundle exec *), Bash(make lint *), Bash(make format *), Bash(make test *), Bash(make typecheck *), Bash(make check *), Bash(python -m pytest *), Bash(poetry run *), Bash(uv run *), Bash(cargo test *), Bash(cargo clippy *), Bash(cargo fmt *), Bash(go test *), Bash(go vet *), Bash(git diff *), Bash(git status *), Bash(git log *)
---

# Dev Workflow

## Usage

```text
/dev-workflow --init         # Project setup (detect check/test commands)
/dev-workflow <task>         # Execute workflow (default)
```

## Prerequisites

- **Reviewer skill** (`reviewer` setting, default: ask-peer): Required for plan/code review. Supported: ask-peer, ask-claude, ask-codex, ask-gemini, ask-copilot. If the configured skill is unavailable, ask user directly instead.
- **extract-rules skill**: Required for rule update. If unavailable, skip with message.

## Configuration

Settings file: `dev-workflow.local.md` (YAML frontmatter only)
- Project-level: `.claude/dev-workflow.local.md` (takes precedence)
- User-level: `~/.claude/dev-workflow.local.md`

```yaml
---
reviewer: "ask-peer"
check_commands:
  - "pnpm run lint:fix"
  - "pnpm run format"
  - "pnpm run typecheck"
test_commands:
  - "pnpm run test:unit"
  - "pnpm run test:e2e"
  - "Skill(test-runner)"
---
```

- **reviewer**: Reviewer skill name (default: `ask-peer`). Choose from: ask-peer, ask-claude, ask-codex, ask-gemini, ask-copilot. Unsupported values fall back to `ask-peer`
- **check_commands**: Static checks (lint, format, typecheck, etc.). Always run all in order
- **test_commands**: Test execution. AI decides whether to run all tests or only related ones based on changes
- Entries starting with `Skill(` are treated as skill invocations (commands and skills can be mixed)
- Executed in array order
- Note: Skills specified with `Skill()` must be installed in the project

## Mode Detection

- `--init` → Init Mode
- Otherwise → Execution Mode

---

## Init Mode

1. Detect project type from config files (package.json, Gemfile, pyproject.toml, Cargo.toml, go.mod, Makefile)
2. Detect package manager from lock files (JS/TS only)
3. Infer check/test commands for the detected project type
   - check_commands: Detect static checks (lint, format, typecheck, etc.)
   - test_commands: Detect test-related keys from package.json scripts, etc. (test, test:unit, test:e2e, test:integration, etc.)
4. Ask user which reviewer skill to use (default: ask-peer)
   - Options: ask-peer, ask-claude, ask-codex, ask-gemini, ask-copilot
5. Present detected commands and reviewer to user for confirmation
6. Save to `.claude/dev-workflow.local.md`

---

## Execution Mode

### Step 1: Load Settings

1. Read `.claude/dev-workflow.local.md` (project-level, priority) or `~/.claude/dev-workflow.local.md` (user-level)
2. If neither exists, prompt user to run `/dev-workflow --init` and stop
3. Resolve `reviewer` from config. If not specified or not in the supported list (ask-peer, ask-claude, ask-codex, ask-gemini, ask-copilot), use `ask-peer`

### Step 2: Create Plan

1. `EnterPlanMode`
2. Analyze the task and codebase, create implementation plan
3. **No code changes in this phase**

### Step 3: Plan Review (max 3 iterations)

1. Call the reviewer skill resolved in Step 1 (e.g. `Skill(ask-peer)`): Review the plan. Instruct reviewer to also read `.claude/rules/` for project conventions.
2. Evaluate feedback, apply improvements, reject inapplicable points
3. If plan was modified, call the reviewer skill again with:
   - the updated plan
   - a summary of changes made in response to the previous feedback
   Count each reviewer call as one iteration, including the initial review. Repeat steps 2-3 until no actionable feedback remains.
4. If no actionable feedback remains, proceed to Step 4. If 3 iterations reached and actionable feedback still remains, present the unresolved points to user for decision.

### Step 4: Finalize Plan

1. `ExitPlanMode` to begin implementation

### Step 5: Implement

1. Follow the plan, track progress with `TodoWrite`

### Step 6: Simplify

1. `Skill(simplify)`: Review changed code for reuse, quality, and efficiency, then fix any issues found

### Step 7: Check / Test (max 3 retries)

1. Run `check_commands` in order (always run all)
   - On failure, fix and retry (do not proceed to test_commands)
2. Run `test_commands` in order
   - Entries starting with `Skill(` are skill invocations; others are shell commands
   - AI decides whether to run all tests or only related ones based on changes (when in doubt, run all)
3. After 3 retries, report to user and stop
4. **Only execute commands/skills from the configuration file**

### Step 8: Code Review (max 3 iterations)

1. Call the reviewer skill resolved in Step 1 (e.g. `Skill(ask-peer)`): Review code changes (include `git diff HEAD` to capture all changes since workflow start). Instruct reviewer to also read `.claude/rules/`.
2. Evaluate feedback, fix genuine issues, reject inapplicable points
3. If code was modified, re-run Step 7, then call the reviewer skill again with:
   - the latest `git diff HEAD`
   - a summary of fixes made in response to the previous feedback
   Count each reviewer call as one iteration, including the initial review. Repeat steps 2-3 until no actionable feedback remains.
4. If no actionable feedback remains, proceed to Step 9. If 3 iterations reached and actionable feedback still remains, present the unresolved points to user for decision.

### Step 9: Update Rules

1. `Skill(extract-rules)` with `--from-conversation` (always)
2. `Skill(extract-rules)` with `--update` (only if significant structural/pattern changes occurred)
3. If extract-rules is unavailable, skip this step and inform user

### Completion

Report summary: tasks completed, files modified, test results, review outcomes, rules updated.
