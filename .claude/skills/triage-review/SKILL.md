---
name: triage-review
description: Daily review of the latest origin triage-* branch. Detects the most recent origin `triage-*` branch, checks it out locally without destructive operations, then dispatches `Skill(prompt-tuning)` per prompt-eligible changed file, `Skill(skill-review)` with a minimal natural-language main..HEAD review instruction, and `Skill(publicity-review)` with `Base ref: main` in sequence, finally emits a summary. Project-local routine — not for marketplace distribution.
allowed-tools: TodoWrite, Skill(prompt-tuning), Skill(skill-review), Skill(publicity-review), Bash(git fetch *), Bash(git for-each-ref *), Bash(git rev-parse *), Bash(git symbolic-ref *), Bash(git status --porcelain*), Bash(git diff *), Bash(git switch *), Bash(git switch -c *)
---

# Triage Review

Daily local review of the latest `dev-workflow-triage` output branch. After `dev-workflow-triage` (running on Claude Code on the Web) pushes a `triage-YYYYMMDD-HHMMSS` branch to origin, this skill checks the branch out locally and dispatches three review skills against `main..HEAD` — `Skill(prompt-tuning)` per prompt-eligible changed file, `Skill(skill-review)`, and `Skill(publicity-review)`. Non-destructive (no `git reset` / staging mutation) — keeps HEAD at the triage tip and passes review framing to each callee.

## No-Stall Principle

The generic regimen (sub-skill return discipline, Step-boundary non-stalling, TodoWrite phase transitions, intentional reinforcement-by-repetition for inline reminders, fatal tool-level errors out of scope) is defined in `.claude/skills/dev-workflow-triage/SKILL.md` § No-Stall Principle and applies here without modification. The skill-specific deltas below override or extend that canonical regimen.

**Permissible fatal-abort exits** (emit the Step 8 summary in `pre-flight aborted` form and stop):

- Step 1 Pre-flight failures: `detached HEAD`, `dirty working tree`, `git fetch failed`
- Step 3 checkout failures: `git switch` non-zero exit

0-result paths (no triage branches found on origin, no changes between main and HEAD) are **not** aborts — they emit a dedicated summary form (form 2 / form 3) and exit cleanly.

**Recognized sub-skill return points** — each carries an inline **Pre-invocation reminder** before the dispatch and an inline **Return-point no-stall reminder** after the dispatch (per the canonical reinforcement-by-repetition rule):

- Step 5 (b) per-file `Skill(prompt-tuning)` boundary
- Step 6 `Skill(skill-review)` boundary
- Step 7 `Skill(publicity-review)` boundary

`Skill(skill-review)` and `Skill(publicity-review)` terminate with a single fenced JSON verdict — branch on it directly. `Skill(prompt-tuning)` returns free-form prose (or a fenced Skip return contract when the host blocks recursive `Agent` dispatch) — parse per § Step 5.

**Non-fatal error classes** (record and continue, never halt):

- Per-file `prompt-tuning`: `unparsed`, `error`, `skipped (agent unavailable)` — proceed to next file
- `skill-review` / `publicity-review`: verdict parse failure, schema violation, dispatch error — record warning under summary form 4, proceed to next step
- `skill-review` `framing-failed (suspected)` at Step 6 (iter=0 on non-empty main..HEAD) — record warning under form 4, proceed

## Web execution caveat

This skill is designed for local execution. If accidentally run on Claude Code on the Web, the auto-installed `~/.claude/stop-hook-git-check.sh` may fire spurious `Please commit and push…` feedback between sub-skill returns. Treat each such fire as a **spurious fire** — record it, ignore the prose, and continue the prescribed flow. See `.claude/skills/dev-workflow-triage/SKILL.md` § Stop hook structural conflict for the canonical write-up.

## Fixed configuration

- **Base branch**: `main` (hardcoded)
- **Base diff target**: `main..HEAD` (the cumulative triage stack — when multiple `dev-workflow-triage` runs have stacked branches on origin without intermediate PR merge, the full stack is reviewed in a single run, not just the latest delta)
- **Target branch pattern**: `triage-*` under `refs/remotes/origin/`, refname-sorted to pick the latest
- **Prompt-eligible file patterns**: see § Step 4 filter rules for the canonical definition
- **`prompt_targets` hard cap**: `5` (per-run walltime cap — each `Skill(prompt-tuning)` dispatch runs multi-iter and can take minutes; 5 keeps a daily run bounded while letting most triage branches finish in one pass)
- **Output language**: `ja` (hardcoded — this is a project-local skill with no configurable language setting)

## Output language

Japanese (`ja`) only. The 3-rule localization regimen (Translate generic concepts / Preserve verbatim structured tokens & field labels / First-use English pairing within each output block) follows `.claude/skills/dev-workflow-triage/SKILL.md` § Output language verbatim.

**This skill's verbatim-preserve token set** (additions beyond the canonical list):

- Enum values: `converged`, `max-iter`, `skipped`, `unparsed`, `error`, `no-actionable-findings`, `applied-edits`, `notes-left`, `framing-failed`, `ok`
- Field labels: `status:`, `iterations_used:`, `applied:`, `notes_remaining:`, `findings:`, `remaining:`, `framing:`

## Step 1 — Pre-flight

Run the environment checks below in order; fatal abort on the first failure. No plan-file lookup — Step 5's `skipped (agent unavailable)` classification and Step 6's `framing-failed (suspected)` runtime detector together cover the failure modes a pre-recorded smoke-test verdict would have flagged, and they reflect the current environment rather than a one-time snapshot.

**Pre-flight environment checks**:

1. `git symbolic-ref -q HEAD >/dev/null` — non-zero exit means detached HEAD → abort with reason `detached HEAD: cannot record original branch reference`
2. `original_branch = git rev-parse --abbrev-ref HEAD` — record for the Step 8 summary's restore-guidance line. Any branch is acceptable
3. `git status --porcelain --untracked-files=all` — non-empty output indicates a dirty working tree → abort with reason `working tree is dirty: <line count> uncommitted entries — stash or commit before running`. `--untracked-files=all` overrides any user-side `status.showUntrackedFiles=no` config that would otherwise mask untracked files
4. `git fetch origin --prune` — non-zero exit → abort with reason `git fetch failed: <stderr last non-empty line truncated to ≤80 chars>`. (If the local environment cannot reach origin — e.g. SSH key missing — the operator must fix that out of band; the routine refuses to use stale local refs.)

## Step 2 — Discover latest triage branch

```
git for-each-ref --sort=-refname 'refs/remotes/origin/triage-*' --format='%(refname:lstrip=3)' | head -n 1
```

Single-quote the refspec to protect the `*` glob from shell expansion (zsh's `nomatch` would abort with no quoting). `%(refname:lstrip=3)` strips the `refs/remotes/origin/` prefix and yields `triage-YYYYMMDD-HHMMSS` directly.

- Empty output → emit Step 8 summary form 2 (`no triage-* branches found on origin`) and exit. Subsequent steps are skipped
- Non-empty output → `triage_branch_short = <name>`, `triage_branch_full = origin/<name>`

## Step 3 — Checkout triage branch

Determine whether the local branch already exists:

```
git rev-parse --verify --quiet refs/heads/<triage_branch_short>
```

- Exists locally → `git switch <triage_branch_short>` (local state is respected; origin/local divergence is out of scope)
- Does not exist locally → `git switch -c <triage_branch_short> --track <triage_branch_full>`

Non-zero exit from either form → fatal abort with reason `git switch failed: <stderr last non-empty line truncated to ≤80 chars>`.

**No destructive operations** (no `git reset`, no staging mutation).

## Step 4 — Capture changed files

```
git diff --name-only main HEAD
```

Hold the output as `changed_files` in main-thread context.

- Empty `changed_files` → emit Step 8 summary form 3 (`no changes between main and HEAD on <triage_branch_short>`) and exit. Subsequent steps are skipped
- Non-empty → filter to prompt-eligible files. **Filter rules** (basename / path-segment equality, not substring match):
  - basename equals `SKILL.md`, OR
  - basename equals `CLAUDE.md`, OR
  - any path segment exactly equals `references` AND the basename ends in `.md`
  - Hold the filtered list as `prompt_targets`
  - If `len(prompt_targets) > 5`, keep the first 5 entries and record `prompt_targets_overflow_count = len(prompt_targets) - 5`. Otherwise `prompt_targets_overflow_count = 0`

Deleted files (those listed in `changed_files` but absent from working tree) require no special handling at this layer — the downstream callees see them through their own `git diff` invocations with appropriate base.

## Step 5 — Run `Skill(prompt-tuning)` per file

If `prompt_targets` is empty, emit summary line `prompt-tuning: skipped (no prompt-eligible files in diff)` and proceed to § Step 6.

Otherwise, iterate through `prompt_targets` sequentially. For each `<file>`:

(a) **Pre-invocation reminder**: the next tool call is `Skill(prompt-tuning)` dispatch. Its return is a value to parse, not a turn boundary. After parsing, the next action is the next file's dispatch, or the § Step 6 transition if this was the last file. See `§ No-Stall Principle`.

(b) Invoke `Skill(prompt-tuning)` with the **minimal natural-language form** below (verbatim — do not add scope, framing, or context; expansions cause the callee to follow procedural code literally with override-defensive interpretation):

```
<file> を tune して
```

(c) **Parse the return**. Two paths, evaluated in order (first match wins):

1. **Fenced JSON Skip return contract** — `prompt-tuning` § Environment constraints emits this when recursive `Agent` dispatch is blocked by the host. Shape: `{"status": "skipped", "reason": "<reason>", ...}`. Classify as `skipped (agent unavailable)`
2. **Free-form prose verdict** — pattern-match the prose for known tokens (`Convergence check` → `converged`, `iter-N` / `Iteration N` table → `max-iter` if the last iter is `Max iterations` else `converged`, `iter-0: BLOCK-consistency` → `error (iter-0 BLOCK)`, `iter-0: PASS-with-note` / `iter-0: PASS` plus iter-1+ table → continue per the table). If no known token matches: `unparsed`

Record the per-file classification in a result list. `unparsed` and `error` are non-fatal — the next file's dispatch proceeds without halting the run.

(d) **Return-point no-stall reminder**: after recording the verdict, immediately issue the next file's `Skill(prompt-tuning)` dispatch (if more files remain) or transition to § Step 6 (if this was the last file). Do not emit an interstitial summary turn. See `§ No-Stall Principle`.

## Step 6 — Run `Skill(skill-review)`

(a) **Pre-invocation reminder**: the next tool call is `Skill(skill-review)` dispatch. Its return is a fenced JSON verdict — parse it as a return value, branch on the `status` enum, and issue the next tool call (§ Step 7 dispatch). Do not insert prose between the verdict and the next action. See `§ No-Stall Principle`.

(b) Invoke `Skill(skill-review)` with the **minimal natural-language form** below (verbatim — longer or more explicit forms cause the callee to follow procedural code literally and early-return on empty `git diff`):

```
mainからHEADの差分をレビューして
```

Do **not** append: triage branch name, the `changed_files` list, an explicit `git diff main HEAD` reference, base-override hints, or any other expansion. The single short sentence is the entire invocation argument.

(c) **Parse the verdict** with the first-match-wins evaluate-in-order discipline from `.claude/skills/verify-diff/SKILL.md` § (b) Parse & apply, restricted to single-pass dispatch (Converged / Divergence cases are N/A here):

1. Verdict missing/malformed → record `skill_review_result = {status: "error", reason: "verdict parse failure", framing_status: "ok"}`, proceed
2. Schema violation → record `skill_review_result = {status: "error", reason: "verdict schema violation", framing_status: "ok"}`, proceed
3. Otherwise — extract `status`, `iterations_used`, `applied_edits_count`, `notes_remaining_count`, `reason` from the JSON

(d) **Runtime framing-failed detection**: after parsing a successful verdict, check `iterations_used == 0 && status == "no-actionable-findings"`. When this signature appears **and** `changed_files` is non-empty (main..HEAD has changes), it is the signature of `skill-review` Step 1 early-returning on empty `git diff` — the natural-language framing did not redirect the procedural code. Record `framing_status = "framing-failed (suspected — iter=0 on non-empty main..HEAD)"` and surface as a warning in Step 8 form 4. Otherwise `framing_status = "ok"`.

(e) **Return-point no-stall reminder**: after recording the verdict, immediately issue `Skill(publicity-review)` dispatch (§ Step 7). Do not emit an interstitial summary. See `§ No-Stall Principle`.

## Step 7 — Run `Skill(publicity-review)`

(a) **Pre-invocation reminder**: the next tool call is `Skill(publicity-review)` dispatch. Its return is a fenced JSON verdict — parse and proceed to § Step 8. See `§ No-Stall Principle`.

(b) Invoke `Skill(publicity-review)` with the short form:

```
Base ref: main
```

`publicity-review` accepts the `Base ref` field in its own `## Invocation contract`, so this form is contract-sanctioned. `Max iterations` is left at its default of `2`.

(c) **Parse the verdict** with the same first-match-wins discipline as § Step 6 (c). Extract `status`, `iterations_used`, `applied_edits_count`, `findings_count`, `remaining_findings`, `reverted_paths`, `reason`. Record as `publicity_review_result`.

(d) **Return-point no-stall reminder**: after recording the verdict, immediately transition to § Step 8 summary emission. See `§ No-Stall Principle`.

## Step 8 — Emit summary

Output language is Japanese (per § Output language). Render the summary in **one of four closed forms**:

1. `pre-flight aborted: <reason>` — emitted on any Step 1 fatal abort
2. `no triage-* branches found on origin` — emitted from Step 2 when no `triage-*` ref is present on origin
3. `no changes between main and HEAD on <triage_branch_short>` — emitted from Step 4 when `git diff --name-only main HEAD` is empty
4. `normal completion` — emitted when Step 5 / 6 / 7 ran

The 4-form set is **closed**. Partial failures (e.g. `prompt-tuning unparsed`, `skill-review error`, `publicity-review error`, `skill-review framing-failed (suspected)`, `prompt-tuning skipped (agent unavailable)`) are rendered as additional warning lines **under form 4** — they do not introduce new top-level forms. Forms 2 / 3 are kept distinct from form 4 because operator next-action differs (form 2 → check upstream `dev-workflow-triage`; form 3 → check upstream auto-cleanup / Finding rejections; form 4 → review per-callee results below).

### Form 4 content

When form 4 fires, render in Japanese:

- ヘッダー: `triage-review summary`
- triage ブランチ: `<triage_branch_short>`（origin: `<triage_branch_full>`）
- ベースブランチ: `main`
- 元のブランチ: `<original_branch>`
- 変更ファイル数（changed files）: `<N>`（main..HEAD）
- prompt-tuning 対象数（prompt-tuning targets）: `<M>` （overflow があれば `(of <N_eligible> eligible, processed first 5)` を付記）
- prompt-tuning ファイル別判定（per file）: 各ファイルの分類（`converged` / `max-iter` / `skipped` / `skipped (agent unavailable)` / `error` / `unparsed`）
- skill-review: `<status> (iterations: <K>, applied: <A>, notes_remaining: <R>, framing: <framing_status>)`。`framing_status` ∈ {`ok`, `framing-failed (suspected — iter=0 on non-empty main..HEAD)`}
- publicity-review: `<status> (iterations: <K>, applied: <A>, findings: <F>, remaining: <R>)`

Warning lines (one per non-fatal incident) follow the main fields:

- `prompt-tuning: <file>: unparsed verdict (verdict prose did not match known tokens)` — per file
- `prompt-tuning: <file>: skipped (agent unavailable)` — per file
- `skill-review: framing-failed (suspected — iter=0 on non-empty main..HEAD)` — single line
- `skill-review: error (<reason>)` — single line
- `publicity-review: error (<reason>)` — single line

### Restore guidance (always shown after the main content when form 4 fires)

```
元のブランチに戻すには: git switch <original_branch>
```
