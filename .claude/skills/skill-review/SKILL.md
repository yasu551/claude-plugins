---
name: skill-review
description: Review uncommitted skill changes against skill-creator best practices and apply improvements. Use this whenever the user asks to "review skills", "check best practices", "improve SKILL.md", or wants a quality check on skill files before committing. Use this when there are uncommitted diffs in SKILL.md, README.md, or references/ files under skills/ or .claude/skills/. This is for reviewing existing skill changes, not creating new skills from scratch.
allowed-tools: Read, Edit, Glob, Grep, Bash(git diff *), Bash(git status *), Bash(git log *), Skill(document-skills:skill-creator)
---

# Skill Review

Review uncommitted skill file changes against skill-creator best practices, then fix what needs fixing.

## Process

1. **Detect changed skill files**: run `git diff --name-only` and `git diff --name-only --cached` to find uncommitted changes. Filter to files matching `skills/*/SKILL.md`, `skills/*/README.md`, `skills/*/references/*`, `.claude/skills/*/SKILL.md`, `.claude/skills/*/references/*`. If no skill files changed, tell the user and stop
2. **Read changed files**: for each changed skill, read the full `SKILL.md` and any changed `references/` or `README.md`
3. **Review via skill-creator**: invoke `Skill(document-skills:skill-creator)` asking it to review the changed skill files against best practices and propose improvements. Pass the file paths and the diff context so it can focus on what changed rather than auditing from scratch
4. **Apply improvements**: fix the issues skill-creator identified. For structural changes (extracting to `references/`, significant rewrites), confirm with the user first
5. **Verify**: re-read changed files to confirm fixes landed correctly

## Scope

- Only review files that have uncommitted changes — diff-scoped, not a full audit
- Project conventions (`.claude/rules/`, `CLAUDE.md`) override best practices if they conflict
- Don't chase perfection — fix real issues, note minor ones, move on
