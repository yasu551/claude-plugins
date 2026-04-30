# Project Rules - Examples

## Principles Examples

### bundle スキルと dev-workflow-bundle のペア bump
**Good** (CHANGELOG.md):
```markdown
## 2026-04-25

### dev-workflow v1.34.2 / dev-workflow-bundle v1.34.2

- fix(dev-workflow): ...
```
**Bad:**
```markdown
## 2026-04-25

### dev-workflow v1.34.2

- fix(dev-workflow): ...
```
（`dev-workflow-bundle` の対が抜けると bundle 配布の version が静かに古いまま残る）

### `Edit` での marketplace.json version 書き換え
**Good:**
```jsonc
// old_string (name の閉じる " と trailing , まで含める)
"name": "dev-workflow",
      "source": "./plugins/dev-workflow",
      "version": "1.34.2",

// new_string
"name": "dev-workflow",
      "source": "./plugins/dev-workflow",
      "version": "1.34.3",
```
**Bad:**
```jsonc
// old_string が "name": "dev-workflow" だけ → "dev-workflow-bundle" の prefix と被って not-unique error
"name": "dev-workflow"
```
Edit 直後に `jq empty .claude-plugin/marketplace.json` で syntax 確認する。`replace_all` は禁止。

### bookkeeping commit の分離
**Good** (commit log):
```text
feat(dev-workflow-triage): release bookkeeping after accepted Findings
fix(dev-workflow): Step 7.5 semantic-judgment branch
chore(release): bump dev-workflow / dev-workflow-bundle (auto-triage 2026-04-25)
```
（per-Finding fix と version bump が別コミット）
**Bad:**
```text
fix(dev-workflow): Step 7.5 semantic-judgment branch + bump version
```
（同じコミットに混ぜると「1 accepted Finding = 1 commit」「scope check」の意味が薄れる）

### Routine スキルの per-invocation 件数 cap heuristic
**Good** (`dev-workflow-triage` SKILL.md Step 2):
```text
gh issue list --repo <owner>/<repo> --state open --limit 50
# Exactly 50 ⇒ overflow=true; report "50-issue cap reached" in Step 4 summary
```
（subagent dispatch コストを考慮して保守的に 50 に設定。cap 到達時は overflow フラグで人に追加 invocation を促す）
**Bad:**
```text
gh issue list --repo <owner>/<repo> --state open --limit 200
# Exactly 200 ⇒ overflow=true
```
（200 件を 1 routine 走行で順次 triage すると verify-diff/skill-review の subagent dispatch が積み重なって walltime が膨らむ）

### 0-item 経路の multi-row flip
**Good** (per-issue ループが 0 件で skip される SKILL.md 記述):
```markdown
If Step 2 reported 0 open issues, mark Step 2 / Step 3 / Step 3.7 / Step 4 all
`completed` in a **single TodoWrite call** and proceed directly to Completion summary.
```
（phase 行を 1 call で同時遷移させて stall 誘発点を減らす）
**Bad:**
```markdown
If 0 issues, mark Step 2 completed.
Then mark Step 3 completed.
Then mark Step 3.7 completed.
Then mark Step 4 completed.
```
（phase 行ごとに別 call にすると、call 間でターン跨ぎの停止誘惑が入る）

### Forward jump pointer for skip path
**Good** (`§ Title match` の skip path 末尾):
```markdown
The per-issue row is flipped pending → completed here per § Title-mismatch skip path.
Skipping does not bypass the reminder dispatch — apply the dispatch at the end of § Close decision.
```
（短絡 path から下流 dispatch への forward jump を明示）
**Bad:**
```markdown
On title mismatch, skip the issue and continue.
```
（「skip = 何もしない」と誤読されると下流の必須 reminder dispatch が抜ける）

### 並列 reminder dispatch（runtime variant 選択）
**Good** (`§ No-Stall Principle` issue-loop boundary):
```markdown
**Reminder #1 (more issues remain)**: Ensure the just-finished per-issue row is
completed (regardless of outcome — accepted, rejected, parse-error, title-mismatch-skip,
any non-error result), then proceed to the next issue with the next tool call.
See § No-Stall Principle.

**Reminder #2 (last issue)**: Ensure the just-finished per-issue row is completed
(regardless of outcome — accepted, rejected, parse-error, title-mismatch-skip,
any non-error result), then proceed to Step 3.7 with the next tool call.
See § No-Stall Principle.
```
（両 variant を並列に prose 記述、closed-list 形式で structural 整合）
**Bad:**
```markdown
After the last issue is processed, jump to Step 3.7.
(For non-last issues, reminder #1 in another section applies.)
```
（dispatch 位置が分散すると agent が決定点で参照しにくくなる）

### `~/.claude/` 配下は Claude config root
**Good** (`dev-workflow-triage` Pre-flight):
```markdown
**Stop hook detection** (Web env observability): run
`jq -r '.hooks.Stop // empty' ~/.claude/settings.json`
and if the output is non-empty, set `stop_hook_present=true` and surface
a warning line in the Step 4 summary. Do not abort.
```
（`~/.claude/settings.json` は Claude Code 標準 config root への参照であり、ユーザー固有の絶対パスではないので SKILL.md にハードコードしてよい）
**Bad:**
```markdown
**Stop hook detection**: run
`jq -r '.hooks.Stop // empty' /Users/alice/.claude/settings.json`
```
（特定ユーザーの `/Users/<name>/...` を埋め込むと配布性が壊れる）

### Cross-skill 構造的衝突の orchestrator + callee documenting
**Good** (orchestrator `dev-workflow-triage/SKILL.md` canonical):
```markdown
## Stop hook structural conflict

**Conflict mechanism**: the per-Finding flow in `§ 3.4 Apply accepted Findings`
runs `(b) Edit → (d) Skill(verify-diff) → (d2) Skill(skill-review) ×3 →
(g) commit`. Each subagent dispatch creates a turn boundary, and uncommitted
state between (b) and (g) is **normal**. The hook fires every boundary.

**Correct behavior**: treat hook feedback as a non-fatal `stop-hook spurious
fire` per § No-Stall Principle — record and continue, do not commit prematurely.
```
**Good** (callee `verify-diff/SKILL.md` short note near `§ Scope check boundary`):
```markdown
> **Stop hook note (Web env)**: if a `~/.claude/stop-hook-git-check.sh` style
> Stop hook is registered, it may fire mid-dispatch with uncommitted-change
> feedback. This is a known structural conflict — see
> `§ Stop hook structural conflict` in the orchestrator (`dev-workflow-triage`).
> Ignore the feedback and continue the prescribed flow.
```
（orchestrator に canonical write-up、callee には 2–3 文の short note + stable heading 参照）
**Bad** (callee 側で canonical を full 再記述):
```markdown
## Stop hook structural conflict
**Conflict mechanism**: per-Finding flow runs (b) Edit → (d) verify-diff → ...
（以下 orchestrator と同一の長文）
```
（callee で full 再記述すると、orchestrator 側の更新が伝播しない／冗長）

### Per-turn environment-induced spurious feedback を non-fatal class に列挙
**Good** (`§ No-Stall Principle` 内):
```markdown
**Non-fatal errors (record and continue):**
- Per-Finding/issue: `comment-failed`, `close-failed`, `commit-failed`
- Per-turn (environment-induced): `stop-hook spurious fire` —
  Web env's `~/.claude/stop-hook-git-check.sh` may inject
  `Please commit and push…` between turns; record and proceed to the
  next prescribed action.
```
（per-Finding と per-turn の disposition class を並列に列挙、disposition は同じだが発生粒度を分けて明示）
**Bad:**
```markdown
**Non-fatal errors:** comment-failed, close-failed, commit-failed
```
（per-turn class が抜けると agent が hook フィードバックを fatal 扱いして即 commit する誤動作経路が開く）

### Callee-side fenced JSON return contract for stall-prone sub-skills
**Good** (callee `skill-review/SKILL.md` 末尾):
````markdown
## Return contract

When invoked as a sub-skill (via `Skill(skill-review)`), terminate the
processing turn by emitting a single fenced JSON block:

```json
{
  "status": "no-actionable-findings" | "applied-edits" | "notes-left" | "error",
  "applied_edits_count": <int>,
  "notes_remaining_count": <int>,
  "reason": "<optional, required when status=error>"
}
```

Do not emit a prose checklist walk-through after the JSON block — the
caller parses the JSON and proceeds. The orchestrator (`dev-workflow-triage`
§ 3.4 Apply accepted Findings) maps `status` to its `record.skill_review`
token and dispatches the next action.
````
（fenced JSON 末尾必須にすると verdict turn が短く閉じて orchestrator の parse → 次 action フローが組める）
**Bad** (callee verdict が free-form prose のみ):
```markdown
After running the checklist:
- Frontmatter: untouched.
- Structure / length: ...
- Writing style: ...
（9 項目の prose checklist walk-through）

No actionable findings.
```
（prose verdict が turn 全体を消費して自然な turn-end を作り、orchestrator の return-point reminder では救えず stall する）

### Orchestrator-side verdict parse-failure handling
**Good** (orchestrator `dev-workflow-triage/SKILL.md` (d2) bullet 内):
```markdown
Parse the fenced JSON verdict at the end of `Skill(skill-review)` output.
Map `status` per the table below; if the JSON block is missing or
malformed (verdict parse failure), or `status: "error"` is returned,
**terminate the (d2) loop with no retry**, increment the (e) error
counter, and proceed to the next Finding.

| status | record.skill_review token | next action |
|---|---|---|
| `no-actionable-findings` | `clean` | proceed to (f) |
| `applied-edits` | `notes left after applied-edits (<n>)` if `notes_remaining_count > 0` else `clean` | proceed to (f) |
| `notes-left` | `notes left after 3 iters (<n>)` | proceed to (f) |
| `error` / parse-failure | `parse-error` | terminate (d2), no retry |
```
（verify-diff の `Verdict missing or malformed` style と整合させ、JSON 経路の `error` と parse-failure を別経路として明文化）
**Bad:**
```markdown
After `Skill(skill-review)` returns, judge the result and proceed to (f).
```
（callee contract が壊れたときの分岐が無く、orchestrator が無限 loop か沈黙に落ちる）

### Aggregate counter warning string differentiation
**Good** (Step 4 aggregate summary):
```text
- skill-review notes left after applied-edits (3)   # M2: applied-edits 経路の残 notes
- skill-review notes left after 3 iters (1)         # M2: iter 上限到達経路の残 notes
```
（同じ `notes_remaining_count` でも sub-condition が違うので warning 文字列を differentiate、SKILL.md にも区別意図を 1 行明記）
**Bad:**
```text
- skill-review notes left (3)
- skill-review notes left (1)
```
（同一文字列に集約すると user が「どの sub-source が threshold を踏んだか」を identify できず、後追い triage で原因切り分け不能）
