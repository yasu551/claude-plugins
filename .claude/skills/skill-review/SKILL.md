---
name: skill-review
description: Review uncommitted skill changes against an internal checklist of skill-creator best practices and apply improvements. Use this whenever the user asks to "review skills", "check best practices", "improve SKILL.md", or wants a quality check on skill files before committing. Use this when there are uncommitted diffs in SKILL.md, README.md, or references/ files under skills/ or .claude/skills/. This is for reviewing existing skill changes, not creating new skills from scratch. Runs standalone — no external skill dependencies.
allowed-tools: Read, Edit, Glob, Grep, Bash(git diff *)
---

# Skill Review

## Process

1. **Detect changed skill files**: run `git diff --name-only` and `git diff --name-only --cached` to find uncommitted changes. Filter to files matching `skills/*/SKILL.md`, `skills/*/README.md`, `skills/*/references/*`, `.claude/skills/*/SKILL.md`, `.claude/skills/*/references/*`. If no skill files changed, tell the user and stop
2. **Read changed files**: for each changed skill, read the full `SKILL.md` and any changed `references/` or `README.md`
3. **Review against the internal checklist**: read `references/best-practices.md`. Walk the checklist and flag items the changed content fails. Only the modified sections of changed files are in-scope (frontmatter fields the diff touched, paragraphs replaced, lines added). Don't audit sibling sections or other files the diff didn't touch. Also flag "hallucination gaps" — points in the changed content where the executing agent would have to guess (ambiguous filenames, unstated success criteria, missing decision rules between branches); these are not on the checklist but are a common failure mode
4. **Apply improvements**: fix the issues identified. Mechanical fixes — rewording a vague description, swapping a `Bash(*)` wildcard for a narrow pattern, trimming heavy-handed MUST phrasing, fixing a broken link — can be applied directly. Confirm with the user first before structural changes: moving content between files, deleting sections, or rewriting large portions of a section. **When invoked as a sub-skill** (no human in the loop — see § Scope's sub-skill no-stall bullet), do **not** wait for confirmation: leave structural changes unapplied and surface them via `notes_remaining_count` in the Return contract verdict instead. Standalone behavior is unchanged
5. **Verify**: re-read changed files to confirm fixes landed correctly. Then emit the fenced JSON verdict defined in `§ Return contract` as the final action of the invocation. **When invoked as a sub-skill, do not produce any further turn after the JSON block** — control returns to the caller, which decides the next action

## Scope

- Only review files that have uncommitted changes — diff-scoped, not a full audit
- Project conventions (`.claude/rules/`, `CLAUDE.md`) override the checklist where they conflict
- Don't chase perfection — fix real issues, note minor ones, move on
- On Claude Code on the Web the auto-installed `~/.claude/stop-hook-git-check.sh` fires on every Stop event and feeds back `Please commit and push…` between Process steps; treat each fire as a **spurious fire** — record it, ignore the prose, and run Process steps 1–5 to completion. Do **not** commit from inside this skill; commit policy lives with the caller. See `dev-workflow-triage` SKILL.md `§ Stop hook structural conflict` for the canonical write-up.
- **Sub-skill no-stall (caller-side note)**: when this skill runs as a sub-skill, structural-change confirmation is suppressed (Process step 4) and the fenced JSON verdict from `§ Return contract` is the terminal output. Diff-scoped file changes are unchanged from standalone — only the user-confirm gate and the post-verdict no-stall discipline differ. The canonical no-stall write-up is `dev-workflow-triage` SKILL.md `§ No-Stall Principle`; see also `§ Return contract`'s *Sub-skill caller directive* for the contract-side restatement.

## Return contract

This skill follows the same **contract pattern** as `verify-diff` § Step 5 — Emit structured summary (a single fenced JSON block at the very end of the invocation). The schema is `skill-review`-specific. Only one fenced JSON block must appear in the response — the verdict block — so callers can locate it unambiguously.

End every invocation with a single fenced JSON block matching this schema:

```json
{
  "status": "no-actionable-findings|applied-edits|notes-left|error",
  "applied_edits_count": N,
  "notes_remaining_count": N,
  "reason": "scope violation|frontmatter broken|null"
}
```

Field semantics:

- `status`:
  - `no-actionable-findings`: the checklist walk produced nothing actionable for the changed-file scope
  - `applied-edits`: at least one mechanical fix was applied this invocation
  - `notes-left`: at least one item was flagged but `applied_edits_count == 0` (e.g. only structural changes were flagged and the sub-skill rule kept them unapplied)
  - `error`: an internal error occurred — see `reason`
- `applied_edits_count`: non-negative integer count of `Edit` calls that successfully landed
- `notes_remaining_count`: non-negative integer. Count of structural / still-actionable items the skill chose not to apply this invocation
  - **Standalone**: always `0`. Structural changes go through the user-confirm dialogue before the verdict; user-declined items are reported in the prose checklist walk above the JSON, not via this counter
  - **Sub-skill mode**: the only path that uses `notes_remaining_count > 0`
  - **`no-actionable-findings` status**: always emits `0` regardless of mode
- `reason`: enum string only when `status == "error"`, otherwise JSON `null`. Keep `reason` payloads to the listed enum tokens — no free-form text, newlines, or control characters — so the verdict stays mechanically parseable

**When to emit `status: "error"`**: the skill itself emits `error` when it detects a problem caused by its own actions before completing — specifically, (i) `reason: "frontmatter broken"` if a re-read after Edit shows the YAML frontmatter no longer parses, or (ii) `reason: "scope violation"` if `git diff --name-only` after Edit lists paths outside the changed-file scope captured at Step 1. In either case, surface the error via the JSON verdict instead of attempting recovery; the caller decides how to handle it. Verdict-block-level failures (caller cannot find or parse the JSON) are caller-side concerns and are not produced by this skill — see the orchestrator's mapping table for that handling.

**Sub-skill caller directive**: when invoked as a sub-skill from `dev-workflow-triage`'s `§ Apply accepted Findings (sub-flow (a)-(g) per Finding)` (d2) bullet, this JSON block is the terminal output of the invocation. Do **not** produce any additional turn after the JSON — the caller's mid-Finding flow continues with sub-step (f) Scope check + stage. See `dev-workflow-triage` SKILL.md `§ No-Stall Principle` for the canonical write-up. Other callers (e.g. `dev-workflow`'s `hooks.on_complete` mechanism) inherit the same emit-and-stop discipline by convention, but the no-stall load-bearing case is `dev-workflow-triage` because that caller is the one that re-engages a queued next sub-step.

## Keeping the checklist fresh

`references/best-practices.md` is a snapshot of upstream `document-skills:skill-creator` guidance — it does not auto-update when the upstream plugin changes. When a meaningful divergence is noticed, refresh this file from the latest skill-creator and ship the refresh as its own commit.
