# Task Decomposition

Deep reference for Step 1.5. Read this when either:

- Normal sub-mode is running and `task_decomposition` is `true` (default), or
- Resume sub-mode is running (`--resume <state-file>`)

If neither applies, you can skip this file — Step 1.5 has no work to do.

`EnterPlanMode` is reserved for Step 2. Any proposal in this step is a plain yes/no dialogue, not a plan.

## State file schema

Every state file lives at `.claude/plans/dev-workflow.<slug>.md` and consists of YAML frontmatter plus a short human-readable body. Validate these invariants on every read (Resume) and match them on creation (Normal) — stop and report the first violation if broken, since a malformed state file would silently corrupt subtask boundaries.

- Required top-level keys: `parent_task` (string), `slug` (kebab-case string), `created_at` (ISO 8601 string), `subtasks` (non-empty array)
- Each subtask has: unique integer `id`, non-empty `title` / `description` / `verification_hint`, `depends_on` (array of prior subtask ids — every referenced id must exist, and the overall graph must be a DAG with no cycles), `status` ∈ {`pending`, `in_progress`, `completed`}, `pr` (string URL or `null`)

**Single-writer constraint**: never run two concurrent `/dev-workflow` sessions against the same state file. Parallel writers race on both the file itself and the shared `git HEAD` that subtasks use as base-commit, silently corrupting subtask boundaries. Different parent tasks (different slugs) in parallel are fine.

## Canonical state-file path

Once a state file is resolved (Resume) or created (Normal), hold its **resolved absolute path** as the canonical state-file path for the rest of the workflow. Every subsequent read, write, and the final delete must target this exact path, never a re-derivation from `slug` — under symlinks, collision-suffixed slugs, or non-standard layouts, the two can disagree and the workflow would orphan or mis-delete state files.

## Parent-task TodoWrite row

When a state file is in play, surface parent progress with a single top-level `TodoWrite` row: `Parent task: <done>/<total> subtasks done — <slug>`. This row is a progress **display**, not a work item — keep its status as `pending` throughout the parent task's lifetime. Never mark it `in_progress`: TodoWrite expects exactly one `in_progress` row at a time, and that slot belongs to whichever workflow step is actually running. Refresh the `<done>/<total>` count whenever a subtask finishes, and remove the row entirely when the state file is deleted. Only insert the row once a state file actually exists (i.e. after Resume step 5 below, or after Normal step 3.f below). A run that stays undecomposed gets no row.

## A. Resume sub-mode (`--resume <state-file>` provided)

1. Resolve `<state-file>` to an absolute path in this order, stopping at the first hit:
   1. An existing file path (absolute or project-relative)
   2. `.claude/plans/<arg>`
   3. `.claude/plans/dev-workflow.<arg>.md`
   4. Not found — stop and list `.claude/plans/dev-workflow.*.md` candidates via Glob
2. Record the resolved absolute path as the canonical state-file path (see above)
3. Read the state file. On YAML parse error, stop and tell the user to back up and manually repair (no automatic recovery — state files are session-scoped scratchpads, and silently rewriting user data is worse than asking)
4. Validate the file against the "State file schema" above. Stop and report the first violation
5. Select the subtask to run:
   - **All subtasks already `completed`**: the previous run finished the last subtask but died before cleanup. Skip straight to the Completion cleanup path — delete the canonical state file, do not add a parent-task TodoWrite row, report the parent task as fully done (with each subtask's title and recorded `pr`, if any), and stop. Do not proceed to Step 2
   - **No leftover `in_progress`** (and at least one `pending`): pick the smallest-id `pending` subtask whose `depends_on` are all `completed`. If none exists (implies a hand-edited file, since a valid DAG always has a runnable frontier while `pending` subtasks remain), stop and report. Otherwise mark the picked subtask `in_progress` and write back
   - **One or more leftover `in_progress`** (interrupted previous session, or hand-edited state file): first re-check each leftover against the runnable condition (all `depends_on` `completed`). Any leftover that is **not** runnable is invalid — reset it to `pending` and do not offer it. Then list the remaining runnable candidates (runnable leftovers + runnable `pending` subtasks) and ask the user which single subtask to run. Reset any unpicked leftover rows back to `pending`, mark the chosen one `in_progress`, and write back. If exactly one runnable candidate exists, keep it as-is without prompting
6. Add the parent-task TodoWrite row (see above). Steps 3–5 must succeed first — if they stopped early, no row is added
7. Summarize to the user: which parent task is resuming, which subtask is current (id, title, description), and its `verification_hint`
8. Set the "effective task" for Step 2 onward to the selected subtask. Keep the full `parent_task` text and the other subtasks' statuses as background context so planning stays consistent with the parent direction
9. Proceed to Step 2

## B. Normal sub-mode (`<task>` provided, no `--resume`)

Prerequisite: this section only applies when `task_decomposition` is `true` (the default). If `false`, Step 1.5 is skipped entirely and you don't need to read this section — the Normal sub-mode request becomes the "effective task" unchanged.

1. Assess whether the task should be decomposed. Keep judgment lightweight and log a one-line rationale:
   - **Do NOT decompose**: single-concern work (typo, config tweak, obvious bug fix, feature contained in one module), or changes that would break consistency if split
   - **Decompose**: multiple independent features, cross-module changes where each module is independently verifiable, and-lists in the request ("implement X and Y and Z"), large refactors that benefit from staged rollout
2. **If "do NOT decompose"**: mark `Step 1.5` as `completed`, set the "effective task" to the original request, and proceed to Step 2 without creating a state file
3. **If "decompose"**:
   a. Draft a subtask list conforming to the "State file schema" above. `verification_hint` describes how completion will be observed (e.g. "migration runs clean", "new auth spec passes", "UI login → logout works end-to-end"). Keep each subtask small enough to ship as a single PR
   b. Validate the draft against the schema (DAG, unique ids, required fields). Revise if invalid
   c. Present the proposal to the user as a plain message (not Plan Mode) and ask: `Proceed with this breakdown into <N> subtasks? (yes / adjust / no = run as one task)`
   d. On "adjust": iterate on the list (add / remove / merge / reorder / edit) and re-validate after each revision
   e. On "no": mark `Step 1.5` as `completed` and proceed to Step 2 as a single undecomposed task (no state file)
   f. On "yes", create the state file and pick the first subtask:
      - Generate a kebab-case `slug` from the parent task (transliterate non-ASCII where reasonable, strip punctuation, lowercase). On collision with `.claude/plans/dev-workflow.<slug>.md` (check via Read or Glob), suffix `-2`, `-3`, ... until free
      - Create `.claude/plans/dev-workflow.<slug>.md` matching the schema (all subtasks `status: "pending"`, `pr: null`) plus a short human-readable body summarizing the breakdown. Create `.claude/plans/` first if missing. Record the created file's absolute path as the canonical state-file path
      - Mark the first runnable subtask (`depends_on: []`, smallest id) as `in_progress` and write back
      - Tell the user the state file path and how to resume it: `/dev-workflow --resume <slug>` or `--resume .claude/plans/dev-workflow.<slug>.md`
      - Add the parent-task TodoWrite row (see above)
4. Set the "effective task" for Step 2 onward to the `in_progress` subtask (or the original request if not decomposed), mark `Step 1.5` as `completed`, and proceed to Step 2

## End-of-run cleanup

The actual cleanup steps that run when a subtask finishes (mark `completed`, record PR URL, pick next subtask, delete state file when done) live in the **Completion** section of `SKILL.md` — that is the single source of truth for completion logic, so no duplicate is kept here to avoid drift. The canonical state-file path you recorded in section A.2 or B.3.f is what the Completion section reads and writes.
