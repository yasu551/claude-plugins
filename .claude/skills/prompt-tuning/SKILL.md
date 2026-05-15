---
name: prompt-tuning
description: Methodology for iteratively improving agent-facing instructions (skills / slash commands / CLAUDE.md / code-gen prompts) by having a bias-free executor run them and evaluating two-sidedly (executor self-report + instruction-side metrics) until improvements plateau. Use after creating or revising a prompt or skill.
allowed-tools: Read, Agent
---

# Empirical Prompt Tuning

The author of a prompt cannot judge its quality. The clearer the writer thinks something is, the more likely another agent will stumble on it. The core of this skill is to **have a bias-free executor actually run the instruction, evaluate it two-sidedly, and iterate**. Do not stop until improvements plateau.

## When to use

- Right after creating or substantially revising a skill / slash command / task prompt
- When an agent does not behave as expected and you want to attribute the cause to ambiguity on the instruction side
- When hardening high-importance instructions (frequently used skills, automation-core prompts)

When not to use:
- One-off throwaway prompts (evaluation cost does not pay off)
- When the goal is not to improve success rate but merely to reflect the writer's subjective preferences

## Workflow

0. **Iteration 0 — description / body consistency check** (static, no dispatch needed)
   - Read the triggers / use cases claimed by the frontmatter `description`. **For targets without frontmatter** (CLAUDE.md fragment, code-gen prompt, free-prose instruction), substitute the **author-provided summary of intent**: a section heading, the leading sentence / paragraph, or — failing both — the first imperative sentence in the body. Note in the iter-0 output which surrogate was used (e.g. `iter-0: PASS (description surrogate: section heading "## Commit messages")`).
   - Read the scope the body actually covers
   - If there is a gap, reconcile description or body before moving to iter 1
   - Example: description says "navigation / form filling / data extraction" but the body is only a CLI reference for `npx playwright test` — detect that kind of gap
   - **Scope**: iter 0 covers description-vs-body **coverage** gaps only. Body-internal defect classes (unresolved placeholders like `<skill-dir>`, dead cross-references, undefined slot-fillings, ambiguous bullets) are **out of scope for iter 0** — note them as iter-N fix candidates and surface them only if the iter 1+ executor actually trips on them. Do not let them block entry to iter 1.
   - **Output shape** (one machine-readable line emitted before iter 1 begins; for **skipped iters** see `## Presentation format` § Skipped-iteration form — the verdict is emitted in exactly one location, immediately above the iter-N table, and that emission satisfies this pre-iter-1 requirement):
     - `iter-0: PASS` — no gap; proceed to iter 1.
     - `iter-0: PASS-with-note (<one-line gap summary>)` — description undersells body, or wording mismatch that does not block a workable contract; record the note, proceed to iter 1.
     - `iter-0: BLOCK-consistency (<one-line gap summary>)` — description claims a capability the body does not cover; reconcile description or body before proceeding. Iter 1 must not run until the verdict is upgraded to `PASS` or `PASS-with-note`.
   - **Isomorphism with the skip return contract's `iter_0` field**: the JSON value of `iter_0` (in the skip return contract defined in `## Environment constraints`) is the same string as this one-liner with the `iter-0: ` prefix stripped — e.g., the one-liner `iter-0: PASS-with-note (description undersells body)` becomes JSON `"iter_0": "PASS-with-note (description undersells body)"`. Use `"not-run"` only when iter 0 itself did not run. Do not re-summarize or shorten when serializing to JSON.
   - If you skip this, the subagent will "reinterpret" the body to match the description, and accuracy will come out high even though the skill does not actually meet the requirements (false positive)

1. **Baseline preparation**: **Freeze** the target prompt (= pin the version under test; do not edit it during this iter's evaluation — "freeze for this round", not "remediate". Remediation happens at Workflow step 5 below). Then prepare the following two things.
   - **Evaluation scenarios**, 2 to 3 kinds. **Composition is hard**: always include at least 1 median (realistic typical use of the target prompt) AND at least 1 edge (atypical / failure-prone case — environment constraint, malformed input, ambiguous request, partial workflow, etc.). **Count is soft**: prefer 3, accept 2 when **executor token cost** is tight (dispatch budget — the main bound; one subagent run ≈ tens of thousands of tokens); never fewer than 2 (one scenario overfits). Note: in non-empirical modes such as baseline-only previews and `## Environment constraints` Alt 3 / structural review where no dispatch occurs, executor cost is zero — count can stay at 3 (or rise if the operator wants broader coverage), bounded only by the operator's own design time. Realistic tasks that assume actual situations where the target prompt would apply.
   - **Designed vs executed scenarios**: the count above fixes the *designed* set at baseline and is reused across iters (trend comparisons need a stable design set). The *executed* set per iter defaults to the whole designed set (dispatch all in parallel per step 2). Under resource bounds, executed may shrink to a strict subset — but the median scenario must execute every iter, and iters with `|executed| < |designed|` are **partial iters**: results inform fixes but do not count toward consecutive-clear judgment in `## Iteration stopping criteria`.
   - **Requirements checklist** (for computing accuracy). For each scenario, enumerate 3 to 7 items the deliverable must satisfy. Accuracy % = items satisfied / total items. Fix this in advance (do not move it afterward).
   - **Target-size scaling** (relaxation of the floors above for tiny targets): the "≥2 scenarios" and "≥3 items per checklist" floors are calibrated for skill-sized targets (~100+ lines, multiple distinct usage paths). For micro-targets they force contrived over-decomposition. Apply this scaling rule when **both** sub-conditions hold:
     - target body ≤ ~50 lines (excluding frontmatter), AND
     - target has ≤ ~3 distinct usage paths / branches (e.g., a 3-command CLI wrapper, a short CLAUDE.md fragment).
     When both hold, the floors relax to **2 scenarios** (still 1 median + 1 edge — composition floor is invariant) and **2 items per checklist** (still ≥1 `[critical]` — success-judgment floor is invariant). When only one sub-condition holds (e.g., short body but many branches), keep the standard floors. When **neither** sub-condition holds (target is large enough AND has rich path-space), keep the standard floors. If the target is even smaller than the relaxation point — body ≤ ~20 lines and ≤ 2 paths — prefer `§ When to use`'s "When not to use" sub-list's "One-off throwaway prompts" branch over applying the methodology at all. Record which regime was applied as a one-line note in the iter-1 baseline-preparation output (e.g., `scaling: tiny-target (relaxed floors to 2 scenarios × 2 items)` or `scaling: standard floors`).
2. **Bias-free read**: Have a "blank-slate" executor read the instruction. **Dispatch a new subagent** via the Agent tool. Do not substitute with a self-reread (it is structurally impossible to view text you just wrote objectively). When running multiple scenarios in parallel, place multiple Agent invocations within a single message.
   - **Pre-flight callability check** (do this once, the first time iter 1 is entered for this run): `allowed-tools: Agent` declares intent but does not guarantee the host actually surfaces the tool (recursive dispatch from a subagent is commonly disabled). Verify Agent callability by checking the **cheapest detection signal first** — do not spend a tool call if registry inspection already answers the question:
     1. **Tool registry inspection** (cheapest, no tool call): inspect the tool surface visible to you, distinguishing two sub-surfaces — the **active surface** (tools enumerated as top-level function definitions, directly invocable without a load step) and any **deferred surface** (lazy-loaded enumeration listing tool names that require a schema-fetch step like `ToolSearch` before invocation). Three sub-states resolve directly:
        - **Absent from both surfaces**: treat as `not callable`, skip steps 2–3 — issuing a dispatch with an absent tool would error without yielding any additional information.
        - **Present only in the deferred surface** (presence-without-load): treat as `not callable` for this run. Hydrating a tool via `ToolSearch` (or equivalent) purely to satisfy a callability probe is a discretionary cost outside this cheapest-first sequence; if the operator wants to actually use `Agent`, they can hydrate it in a run where dispatch is the planned action.
        - **Present in the active surface**: proceed to step 2 (trial dispatch).
     2. **Trial dispatch** (only if step 1 says present): issue the first scenario's dispatch normally. If the call returns a permission-denied / recursion-blocked error rather than a normal subagent reply, treat as `not callable`.
     3. **Silently-empty subagent** (only if step 2 returned): if the dispatch returned but the subagent's body is empty or the `<usage>` block is missing entirely, treat as `not callable` and do not retry — this signals the host accepted the call but did not actually run a subagent.
     If `not callable` by any of (1) / (2) / (3), immediately route to the `## Environment constraints` section and emit the skip return contract defined there — do **not** retry with self-reread.
3. **Execution**: Hand the subagent a prompt that follows the **subagent invocation contract** described below, and have it execute the scenario. The executor produces an implementation or output and returns a self-report at the end.
4. **Two-sided evaluation**: Record the following from the returned results.
   - **Executor self-report** (extracted from the body of the subagent's report): unclear points / discretionary fill-ins / places where template application got stuck
   - **Trace interpretation**: each unclear point is tagged with the phase it originated in (Understanding / Planning / Execution / Formatting — see "Subagent invocation contract"). Phase-local fixes land better than global "the prompt was unclear" fixes; a single Understanding-phase ambiguity often looks like a chain of Execution-phase failures.
   - **Structured reflection**: each unclear point must be returned as `Issue / Cause / General Fix Rule`. The `General Fix Rule` is the class-level abstraction that feeds the "Failure pattern ledger" — without it, fixes stay as one-off patches that rediscover the same mistake later.
   - **Instruction-side measurements** (the judgment rules are defined canonically in this section; refer to it from elsewhere):
     - Success/failure: counts as success (○) only when **all** requirements tagged `[critical]` are ○. If even one is × or partial, it is failure (×). The label is the binary ○ / × only.
     - Accuracy (achievement rate of the requirements checklist, %. ○ = full score, × = 0, partial = 0.5; sum and divide by total items)
     - Step count (the `tool_uses:` value emitted in the `<usage>...</usage>` block at the tail of the Agent tool's return text. Include Read / Grep, do not exclude them)
     - Duration (the `duration_ms:` value in the same `<usage>` block)
     - Retry count (how many times the subagent redid the same decision. Extract from the subagent's self-report; not measurable from the instruction side)
     - **On failure, add a one-line note to the "unclear points" section of the presentation format stating "which [critical] item dropped"** (for root cause tracing)
   - The requirements checklist must include **at least one** `[critical]`-tagged item (if there are zero, the success judgment becomes vacuous). Do not add or remove [critical] tags after the fact.
5. **Apply the diff**: Put the minimum fix into the prompt to eliminate the unclear points. One theme per iteration (multiple related fixes are OK, unrelated fixes go to next time).
   - **Before applying the fix, explicitly state "which item in the requirements checklist / judgment wording this fix satisfies"** (fixes inferred from axis names often do not land. See the "Fix propagation patterns" section below.)
   - **Consult the failure pattern ledger first**. If the structured reflection's `General Fix Rule` already matches a known pattern, the first question is "why didn't the existing fix prevent it?" — the fix may need to move closer to the top of the prompt, or be re-worded, before a new ledger entry is added.
6. **Re-evaluate**: Run 2 → 5 again with a new subagent (do not reuse the same agent: it has learned the previous improvements). Increase parallelism if iterating further does not plateau improvements.
7. **Convergence check**: The rough rule is "stop when 2 consecutive iterations have zero new unclear points AND metric improvements fall below the thresholds defined in the 'Iteration stopping criteria' section". Make it 3 consecutive for high-importance prompts (see that section's Convergence bullet).

## Evaluation axes

| Axis | How to capture | Meaning |
|---|---|---|
| Success/failure | Did the executor produce the intended deliverable (binary) | Minimum bar |
| Accuracy | What % of requirements the deliverable satisfies | Degree of partial success |
| Step count | Tool-call / decision-step count used by the executor | Indicator of instruction waste |
| Duration | Executor's duration_ms | Proxy indicator of cognitive load |
| Retry count | How many times the same decision was redone | Signal of instruction ambiguity |
| Unclear points (self-report) | Executor enumerates as bullets | Qualitative improvement material |
| Discretionary fill-ins (self-report) | Decisions not fixed by the instruction | Surfaces implicit specification |

**Weighting**: Qualitative (unclear points / discretionary fill-ins) is primary, quantitative (time / step count) is auxiliary. Chasing only time reduction makes the prompt too thin.

### Qualitative interpretation of `tool_uses`

Looking only at accuracy hides skill problems. Using `tool_uses` as a **relative value across scenarios** reveals structural defects:

- If one scenario is **3-5x or more** vs the others, that skill is a sign of being **decision-tree-index-leaning with low self-containment**. The executor is being forced into references descent.
- Typical example: all scenarios have `tool_uses` of 1-3 but one scenario alone has 15+ → there is no recipe for that scenario in the skill itself, so it is cross-searching references/
- Countermeasure: adding an "inline minimum complete example" or "guidance on when to read references" at the top of SKILL.md in iter 2 significantly drops `tool_uses`

Even at 100% accuracy, a skew in `tool_uses` is grounds for triggering iter 2. "Cut off based on accuracy alone" tends to miss structural defects.

### Fix propagation patterns (conservative / overshoot / zero-shoot)

Fix → effect is not linear. Pre-estimation can play out in the following 3 patterns:

- **Conservative swing** (estimate > actual): one fix aimed at multiple axes but only moved one. "Aiming at multiple axes tends to miss."
- **Overshoot** (estimate < actual): one structural piece of information (e.g., a combination of command + config + expected output) satisfied judgment wording across multiple axes at once. "Combinations of information structurally hit multiple axes."
- **Zero-shoot** (estimate > 0, actual = 0): a fix inferred from the axis name did not reach any of the judgment wording. "Axis names and judgment wording are different things."

To stabilize this, **before applying the diff, have the subagent verbalize "which judgment wording this fix satisfies"**. Estimation accuracy does not come out unless you tie things at the threshold-wording level. When adding a new evaluation axis, also concretize the judgment criteria for each point down to the threshold-wording level (at a granularity the subagent can judge, such as "all explicit" or "full text of a minimum working configuration" — so it knows what constitutes 2 points).

## Subagent invocation contract

The prompt given to the executor takes the following structure. This is the input contract for "two-sided evaluation".

```
You are an executor reading <target prompt name> with a blank slate.

## Target prompt
<Default: specify an absolute file path the executor will `Read` (path-by-Read is preferred — it avoids context bloat and keeps the executor reading the same source-of-truth the author edits). Paste the full body inline only when the target has no canonical file location (ephemeral / not-yet-saved prompt) AND is short enough that inlining is cheaper than a Read (rule of thumb: < ~30 lines).>

## Scenario
<One paragraph setting the scenario context>

## Requirements checklist (items the deliverable must satisfy)
1. [critical] <item that belongs to the minimum bar>
2. <normal item>
3. <normal item>
...
(Judgment rules are canonically defined in the "Workflow" section's "Two-sided evaluation — Instruction-side measurements" bullet. At least one [critical] is required; there is no upper bound — if every requirement is genuinely on the minimum bar, marking all of them `[critical]` is valid. The 1-critical-plus-normals mix in the example above is illustrative, not prescriptive.)

## Task
1. `Read` the target prompt **once** at the start; do not re-`Read` it unless the prompt body explicitly instructs you to. Re-reads inflate `tool_uses` and corrupt the instruction-side measurement.
2. Follow the target prompt to execute the scenario and produce the deliverable.
3. On completion, respond with the report structure below.

## Report structure
- Deliverable: <artifact or execution summary>
- Requirement achievement: ○ / × / partial (with reason) for each item
- **Trace** (tag OK / stuck / skipped for each phase, one-line reason when not OK):
  - Understanding (reading the instruction and building a mental model)
  - Planning (deciding the approach / ordering)
  - Execution (actually doing the work)
  - Formatting (shaping the deliverable to the expected form)
  - *Collapsed form allowed*: when all four phases are OK, a single line `Trace: all OK` is sufficient. Emit phase-by-phase only when any phase is stuck or skipped. (This avoids happy-path boilerplate; the trace structure only earns its cost when something actually goes wrong.)
- **Unclear points (structured)**: for each issue, three lines:
  - Issue: <what observably happened>
  - Cause: <why, diagnosed at the instruction level>
  - General Fix Rule: <a class-level rule, not a spot fix, that would prevent this class of mistake>
- Discretionary fill-ins: places not fixed by the instruction and filled in by your own judgment (bullets)
- Retries: number of times you redid the same decision and why
```

The caller extracts the self-report portion from the report, then parses the `<usage>...</usage>` block at the tail of the Agent tool's return text for `tool_uses:` / `duration_ms:` to fill the corresponding rows of the evaluation-axis table.

## Environment constraints

In environments where dispatching a new subagent is not possible (already running as a subagent, Agent tool is disabled, etc.), **do not run the empirical loop** (Workflow steps 2–6 require fresh subagent dispatch; without it, "two-sided" collapses into self-reread and the evaluation cannot be trusted). The empirical-loop ban does **not** mean the whole skill is off-limits — the following static-only modes remain available:

- **Alternative 1 — Delegate**: ask the parent session's user to start a separate Claude Code session and delegate the evaluation there.
- **Alternative 2 — Skip with explicit report**: report to the user "empirical evaluation skipped: dispatch unavailable" and stop.
- **Alternative 3 — Structural review mode** (see below): a sanctioned static review that does *not* claim to be empirical.
- **NG**: substitute with a self-reread inside the same agent (bias enters; the result must not be trusted).

**Skip return contract** (machine-parseable, so callers / orchestrators can route deterministically when prompt-tuning is invoked as a sub-skill):

```json
{
  "status": "skipped",
  "reason": "dispatch unavailable" | "user-elected-skip" | "structural-review-only",
  "iter_0": "PASS" | "PASS-with-note (<gap>)" | "BLOCK-consistency (<gap>)" | "not-run",
  "designed_scenarios_count": <int, 0 if baseline-prep (Workflow Step 1) did not complete (e.g., iter 0 BLOCK-consistency, or aborted mid-Step-1); otherwise the count of scenarios established at baseline-prep>,
  "executed_scenarios_count": 0,
  "alternative_taken": "Alt 1 — Delegate" | "Alt 2 — Skip" | "Alt 3 — Structural review"
}
```

Emit this fenced JSON block as the final element of the response whenever the empirical loop is skipped. Iter 0 may still have run and contributed its verdict into the `iter_0` field. The caller branches on `status == "skipped"` and `alternative_taken` to retry in a different environment, escalate, or absorb the skip. **Caller-mandate conflict**: even if the caller demands an empirical iter, the `Agent`-dependent branch cannot honor it under dispatch-unavailable — emit the skip contract above. Do **not** invent simulated dispatch results to satisfy the caller; that silently violates the bias-free-executor premise. The caller may *request* an empirical iter but cannot override the bias-free-executor requirement.

**Selection criteria** (when to pick which):
- **Step 0 — scope filter** (apply before the preferences below): drop any alternative made vacuous by the caller's task framing. Example: when the caller explicitly asked you to "apply the prompt-tuning methodology to <target>", Alt 2 (skip-and-stop) is vacuous — picking it would fail the request by definition. Apply the preferences below only to surviving alternatives.
- Prefer **Alt 1** when the target prompt is high-importance AND the user can readily start a separate session — empirical signal is worth the round-trip.
- Prefer **Alt 2** when (a) the target is large / opaque so even structural review yields little signal, OR (b) the user explicitly wants a "no eval, just ship" answer.
- Prefer **Alt 3** when the target is short enough that static consistency / clarity review yields meaningful textual findings on its own (rule of thumb: fits on one screen, ≤ ~80 lines) AND empirical evaluation will not happen soon. Alt 3 can also chain with Alt 1 as "do structural review now, then run empirical later in a separate session".
- **Threshold tolerance band for the `≤ ~80 lines` rule** (resolve borderline cases without inventing a tiebreaker): the `~80` figure is a rule of thumb, not a hard cutoff. Apply the following bands:
  - **≤ 80 lines**: clear Alt 3 preference (subject to other criteria above).
  - **81 – 120 lines (tolerance band)**: still take Alt 3 if **any one** of these holds — (a) the caller's task framing makes Alt 2 vacuous (i.e., Alt 3 is the only surviving alternative after Step 0 scope filter), (b) the body is well-sectioned (clear `##`-level structure rather than dense reference prose), or (c) the target's iter-0 verdict suggests the description/body gap is the primary axis of interest (which a static review handles well). Otherwise Alt 2.
  - **> 120 lines**: clear Alt 2 preference (static review of a large body yields too little signal).
  - Count *content lines only* (exclude frontmatter and blank lines) when applying these bands. Record the line count and band decision in a **preamble paragraph immediately above the `## Iteration N` heading** (same location as the `Mode: <empirical|Structural review>` declaration per `## Environment constraints` § Report shape under structural review mode); the contract JSON schema is fixed and does not carry these values.
- **Non-interactive default** (no user is available to consult, e.g., single-shot routine execution): default to **Alt 3** when the target ≤ ~80 lines; otherwise default to **Alt 2** with an explicit "skipped: dispatch unavailable, target too large for static review" report. Do not default to Alt 1 in non-interactive runs (it requires user action that cannot be solicited).
- **Detecting "non-interactive" in observable terms** (do not rely on executor introspection — use these signals): treat the run as non-interactive when **any one** of the following holds:
  1. you were invoked as a subagent (via `Agent` / `Task` dispatch) by a parent skill or routine, with no `<user_message>`-style turn possible in your own context;
  2. your runtime context lacks a foregrounded human conversation (no slash-command-from-user, no chat thread you are responding into);
  3. the caller's prompt explicitly framed the run as routine / batch / scheduled execution.
  Otherwise treat the run as interactive (user can be addressed via a normal text response and is expected to read it).

**Structural review mode**: when you want to check only the **consistency and clarity of the description** of the skill / prompt rather than run empirical evaluation, carve it out explicitly. Note clearly in the request prompt to the subagent (or in your own output, when no dispatch is possible) "this round is structural review mode: text consistency check, not execution". Structural review is an aid to empirical, not a replacement: (a) it cannot be used for consecutive-clear convergence judgment, and (b) its findings are **not** entered into the Failure pattern ledger — the ledger is for empirical iterations only.

**Report shape under structural review mode**: structural review borrows the Subagent invocation contract's report skeleton, but execution-bound fields are remapped — apply the table below:

| Report field | Empirical mode | Structural review mode |
|---|---|---|
| Deliverable | execution artifact + summary | the static review findings (description/body gap notes, ambiguity list) **about the target prompt** — this is the primary output in structural review mode |
| Requirement achievement (○ / × / partial per item) | per the iter checklist | per the structural-review checklist passed in |
| Trace (Understanding / Planning / Execution / Formatting) | tag each phase | collapse to `Trace: all OK (no execution)` |
| Unclear points (Issue / Cause / General Fix Rule) | ambiguities in the **prompt-tuning skill itself** that forced the executor into a discretionary call | same — ambiguities in the **prompt-tuning skill itself**, **not** findings about the target prompt (target findings live in `Deliverable`). Same Issue / Cause / General Fix Rule structure. |
| Discretionary fill-ins | choices not fixed by the **prompt-tuning skill** that the executor had to make | same |
| Retries | execution retries | always 0 (no execution) |

Empirical-only metrics (success/failure on execution scenarios, `tool_uses`, `duration_ms`) are **N/A** under structural review and should be omitted from the Presentation-format execution table, not faked.

**Precedence when a caller-supplied report template conflicts with this remap**: the caller's invocation prompt often re-states the Subagent-invocation-contract Report structure verbatim, including empirical-mode fields (Trace phases, Retries, etc.). When the resolved mode is Structural review, this remap table takes precedence over the caller's inlined template — collapse / N/A / omit per the table above, regardless of what the caller's template shows. State once at the top of the report which mode is resolved (e.g. "Mode: Structural review — empirical fields remapped per § Environment constraints"); the caller can then read the rest unambiguously.

**Domain discipline** (applies in both modes): `Unclear points` and `Discretionary fill-ins` always report ambiguities / unfixed choices in the **prompt-tuning skill itself** (the methodology being applied), never findings about the target prompt being evaluated. Target findings flow into `Deliverable` (in empirical mode as part of the execution artifact; in structural review mode as the primary output). Mixing the two domains makes it impossible to tell "is the prompt-tuning skill unclear?" from "is the target prompt unclear?", which defeats the methodology's improvement loop.

## Iteration stopping criteria

- **Convergence (stop)**: 2 consecutive rounds (3 for high-importance prompts, per Workflow's "Convergence check" step) satisfying **all** of the following:
  - New unclear points: 0
  - Accuracy improvement vs previous: +3 points or less (saturation such as 5% → 8%)
  - Step count variation vs previous: within ±10%
  - Duration variation vs previous: within ±15%
  - **Sentinel handling for the two variation checks**: rows whose `steps` or `duration` cell is `—` (per `## Presentation format`'s per-cell sentinel rule) are excluded from the corresponding ±10% / ±15% calculation for that row. The iter still counts toward consecutive-clear judgment if at least the median scenario row has measured values; if every row in the iter is `—`, the iter does not count.
  - **Overfitting check**: at convergence judgment, add 1 hold-out scenario not used so far and evaluate. If accuracy drops 15 points or more from the recent average, overfitting. Go back to baseline scenario design and add edges.
- **Divergence (suspect the design)**: if new unclear points do not decrease across 3+ iterations → the design direction of the prompt itself may be wrong. Stop fixing by patches and rewrite the structure
- **Resource cutoff**: stop when importance and improvement cost no longer balance (the "ship at 80 points" call)

## Failure pattern ledger

Maintain a cumulative list of failure modes across iterations. Without it, each iteration re-discovers the same class of mistake, and accuracy improvements stall without the operator noticing that the same `General Fix Rule` keeps surfacing under different surface wording.

Entry format:

```
- **Pattern name**: short descriptive handle (not "ambiguous X"; prefer "over-eager template application when skip clause is absent")
  - Example: <representative Issue wording from some iter>
  - General Fix Rule: <the class-level rule from that iter's structured reflection>
  - Seen in: iter N, iter M, ...
```

Rules:
- Before generating a fix in Workflow step 5, scan the ledger. If the current `General Fix Rule` matches an existing entry, update `Seen in` and investigate why the existing fix did not prevent recurrence (wording ambiguity? position too late in the prompt? missing example?) before creating a new entry.
- A pattern that recurs 3+ times despite targeted fixes is a structural signal — escalate to the "Divergence" criterion above rather than continuing to patch.
- The ledger is per-target-prompt, not global across all empirical-prompt-tuning runs.

## Variant exploration (optional, plateau-breaking)

When iterations approach a plateau but convergence criteria (2 consecutive clears) are not met, suspect local optimum and run a 2-variant round:

- **Conservative variant**: current prompt + next-best minor fix
- **Exploratory variant**: current prompt with one structural change — reorder sections, split a dense paragraph, drop a redundant section, or add a missing scaffolding (e.g., a worked example)

Dispatch fresh subagents on the same scenarios in parallel (one message with multiple Agent tool calls). Keep the variant with higher accuracy; on tie, prefer fewer unclear points; on further tie, prefer lower `tool_uses`.

Pairwise-comparison caveats:
- Do **not** ask a subagent to rate "A vs B" directly. LLM position bias and self-preference bias make such judgments noisy at small n.
- Compare on the objective axes only (accuracy, step count, unclear-points count, phase-weakness counts). Those are reproducible; "which prompt felt better" is not.
- If qualitative comparison is genuinely needed, counterbalance: run both orderings (A,B) and (B,A) and accept a verdict only if both orderings agree.

Cost: variant exploration doubles dispatch count per iteration. Use when plateau is suspected, not by default.

## Presentation format

**Render rule for plan-only / baseline-only previews**: when an iteration block accompanies a baseline-only deliverable (no execution this round — e.g. the operator intentionally stops at Step 1 baseline preparation to stage scenarios across runs, or wants to publish the iter-1 plan for review before paying dispatch cost), keep the execution-results table present but mark every cell as `(pending — not dispatched)`. Do not silently omit the table. The Structured reflection / Discretionary fill-ins / Ledger updates / Next fix proposal sections are still populated normally, but their **content origin is redirected** for this render: `Structured reflection` sources from iter-0 static findings and baseline-design observations (rather than executor self-reports, which do not exist yet); `Discretionary fill-ins` sources from baseline-design choices (scenario count, edge flavor, draft-prompt selection, etc.); `Ledger updates` is typically empty (no empirical findings to enter; the ledger is empirical-only). When dispatch finally happens in a later iter, these sections revert to their canonical content origins. This mode is distinct from Structural review mode (see Environment constraints): there, dispatch is *impossible*; here, dispatch is *deferred by choice*.

**Per-cell sentinel for partial-iter rows**: when a dispatch was attempted this iter but did not produce metrics for some scenarios (e.g., one scenario timed out, was excluded for resource bounds per Workflow Step 1's `|executed| < |designed|`, or returned without a parseable `<usage>` block), use `—` (em-dash) in the affected cells. This is distinct from the iter-wide `(pending — not dispatched)` form above: `—` is per-cell within an otherwise-dispatched iter. Rows that contain `—` in `steps` or `duration` are **excluded** from the `Step count variation ±10%` / `Duration variation ±15%` calculations in `## Iteration stopping criteria` (per-row exclusion; the iter still counts toward consecutive-clear judgment as long as the median scenario row has measured values). Do not substitute `0`, `N/A`, blank, or invent estimates — only `—` is parseable as "intentionally unmeasured for this cell".

Record and present to the user with the following form at each iteration:

```
## Iteration N

### Changes (diff from previous)
- <one-line fix content>
- Pattern applied: <pattern name from ledger, or "(new)">

### Execution results (per scenario)
| Scenario | Success/Failure | Accuracy | steps | duration | retries | Weak phase |
|---|---|---|---|---|---|---|
| A | ○ | 90% | 4 | 20s | 0 | — |
| B | × | 60% | 9 | 41s | 2 | Execution |

### Structured reflection (newly surfaced this time)
- <Scenario B>: [critical] item N is × — <one-line reason for drop>
  - Issue: <what observably happened>
  - Cause: <why, at the instruction level>
  - General Fix Rule: <class-level abstraction>
- <Scenario A>: (nothing new)

### Discretionary fill-ins (newly surfaced this time)
- <Scenario B>: <fill-in content>

### Ledger updates
- Added: <pattern name> (from Scenario B)
- Re-seen: <pattern name> (originally iter K) — existing fix did not prevent recurrence because <reason>

### Next fix proposal
- <one-line minimum fix>

(Convergence check: X consecutive clears / Y rounds remaining to stop condition)
```

**Skipped-iteration form** (when the *entire* iter is skipped per `## Environment constraints`, not the per-cell case covered by `Per-cell sentinel for partial-iter rows` above): emit the `## Iteration N` heading, fill every metric cell with `—`, write the iter 0 verdict above the table (or `not-run` when iter 0 itself was skipped) — this **replaces, not supplements**, any earlier iter-0 emission required by `Workflow` Step 0; emit the verdict line in exactly one location for a skipped iter, immediately above the table — insert `> **Iter skipped** — see the skip return contract at the end of the response for `alternative_taken` and `reason`.` as a blockquote between the iter-0 verdict line and the table (this disambiguates the all-`—` table from a partial-iter case where every per-cell sentinel happens to be `—`), and append the fenced skip return contract from `## Environment constraints` as the **final element** of the response. Skipped iters do not count toward consecutive-clear judgment in `## Iteration stopping criteria`.

**Other sections under Skipped-iteration form** (disposition of each presentation-format section listed in the template above):
- `Changes (diff from previous)`: keep — record "(no diff applied; iter skipped)" plus the `Pattern applied` line if a known pattern from the ledger motivated the iter's existence; otherwise `Pattern applied: (n/a — skipped)`.
- `Execution results (per scenario)`: keep the table, every metric cell `—` per the rendering rule above.
- `Structured reflection (newly surfaced this time)`: keep — if Alt 3 (Structural review) is the alternative taken, source from the static review's unclear-points about the prompt-tuning skill itself (per `## Environment constraints` § Domain discipline); if Alt 1 / Alt 2 with no static review, this section is empty (`(no reflection — iter skipped without static review)`).
- `Discretionary fill-ins (newly surfaced this time)`: same origin rule as Structured reflection — only present when static review supplied them; otherwise omit the section header.
- `Ledger updates`: always empty (`(no updates — ledger is empirical-only per § Failure pattern ledger)`).
- `Next fix proposal`: present **only** when a static review (Alt 3) produced a targeted improvement candidate for the prompt-tuning skill itself or for the target prompt; otherwise omit. Skipped-iteration form is not a vehicle for fixes derived from non-empirical introspection of the canonical methodology; do not fabricate proposals just to fill the slot.
- **Pre-skip baseline-prep artifacts** (designed scenarios + their requirements checklists from Workflow Step 1, produced *before* the skip decision fires): these are not lost on skip. Default disposition — render them inline under a sub-heading `### Designed scenarios (carried over to next iter)` immediately before the Execution results table, so the operator can audit the design without scrolling into the JSON contract. Under Alt 3, the scenarios may *also* be referenced from the `Deliverable` (when the static review of the target leans on scenario framing) — that is duplication of reference, not duplication of content; the canonical render stays in the `Designed scenarios` block. Under Alt 1 / Alt 2 with no static review, still emit the block (it documents what would have run had dispatch been available). **Degenerate case — baseline-prep did not run** (Step 0 scope filter / Pre-flight callability check resolved to skip before `Workflow` Step 1 entered, so `designed_scenarios_count = 0`): keep the `### Designed scenarios (carried over to next iter)` sub-heading and replace its contents with a one-line note `(none — baseline-prep did not run; skip decision fired before Step 1)`. Do **not** omit the heading silently — preserving it documents that the absence is intentional, not a render bug.
- Convergence-check trailer line: emit `(Convergence check: <X> consecutive clears / iter skipped, does not advance)` so the convergence ledger stays auditable.

## Red flags (beware of rationalization)

| Rationalization that surfaces | Reality |
|---|---|
| "Rereading it myself has the same effect" | You cannot view text you just wrote "objectively". Always dispatch a new subagent. |
| "One scenario is enough" | One scenario overfits. Minimum 2, ideally 3. |
| "Zero unclear points once, so we're done" | Could be coincidence. Finalize with 2 consecutive rounds. |
| "Let's knock out multiple unclear points at once" | You lose track of what worked. One theme per iteration. |
| "Split each related micro-fix strictly into its own iter" | Trap in the opposite direction. "One theme" is a semantic unit. 2-3 related micro-fixes can be bundled into 1 iter. Splitting too far explodes the iter count. |
| "Metrics are good, so ignore qualitative feedback" | Time reduction can also be a sign of being too thin. Keep qualitative primary. |
| "Rewriting from scratch is faster" | Correct if unclear points do not decrease across 3+ iterations. Before that stage, it is escape. |
| "Let's reuse the same subagent" | It has learned the previous improvements. Always dispatch a new one. |

## Common failures

- **Scenario too easy / too hard**: neither produces signal. One at the median of real use, one edge
- **Only looking at metrics**: chasing only time reduction strips important explanations and makes it fragile
- **Too many changes per iteration**: you can no longer trace "which fix back then worked". One theme per iteration (2-3 related micro-fixes can be bundled — see Workflow's "Apply the diff" step and the Red flags table)
- **Tuning scenarios to match the fix**: making the scenario side easier just to make unclear points look eliminated → putting the cart before the horse

## Related (external, optional)

The following are external skills / patterns — they are **not bundled** with this skill. Install separately if available in your marketplace; otherwise treat them as prior art for context, not as required dependencies.

- `superpowers:writing-skills` — the TDD approach for skill creation. Essentially the same as this skill's "baseline → fix → rerun with a subagent"
- `retrospective-codify` — fixating learnings after a task. This skill is during prompt development, retrospective-codify is after a task ends; use them differently
- `superpowers:dispatching-parallel-agents` — conventions for running multiple scenarios in parallel
