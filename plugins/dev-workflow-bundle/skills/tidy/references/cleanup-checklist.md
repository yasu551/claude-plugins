# Cleanup Checklist

The reviewer walks each CHANGED FILE against the items below. Classify each finding as `mechanical_edit` (textual replacement) or `structural_note` (needs human moving / deleting / large rewrite).

Each item is **judgment-style**, not a regex — describe the cleanup opportunity in your own words on the way to producing a fix.

## Preserve functionality (hard constraint, applies before every item)

**Never change what the code does — only how it does it.** All original features, outputs, observable behaviors, error conditions, and side-effect ordering must remain intact after the cleanup. If a candidate fix would alter any of these, it is **out of scope** for this skill regardless of which item below it matches — surface it as a `structural_note` for human review rather than emitting it as a `mechanical_edit`. When unsure whether a fix preserves behavior, default to `structural_note`.

## Items

1. **Redundancy / duplication** — same logic appearing at 2+ sites within the diff (extract or inline); the same data structure declared in multiple places (consolidate).

2. **Dead code** — unused imports, unused variables / parameters, unreachable branches, code that the user explicitly removed earlier in the session and a later edit re-introduced (deletion is authoritative, not a gap to fill).

3. **Over-abstraction / premature generalization** — a helper called from exactly one site, a class with exactly one instantiation, a parameter that always receives the same value. **Counter-rail**: do not delete an abstraction that is earning its keep at multiple call sites or expressing a domain concept — flag only abstractions whose single observed use shows the generalization never materialized.

4. **Defensive guards on already-safe paths** — null checks where the call site guarantees non-null, redundant guard layers where an upstream guard already handles the case (double-coverage).

5. **Speculative features** — functionality beyond the stated requirement; features added "for future use" without an explicit trigger (user requirement, known bug, documented rule).

6. **Comment narration / preamble + naming clarity** — line-by-line paraphrase of code, restating surrounding context, comments that explain *what* the code does instead of non-obvious *why*. Also: identifier names whose meaning is genuinely unclear to a reader unfamiliar with the file (a name that requires the explanatory comment to make sense — fix the name so the comment becomes redundant, then the comment is a delete-candidate).

7. **Redundant prose in docs / SKILL.md** — re-statement of what well-named identifiers already convey, repeated rationale that adds no new constraint, paragraphs that paraphrase an adjacent paragraph.

8. **Naming consistency drift (within-diff scope)** — a rename that did not propagate to all call sites within the same change; a function whose name no longer matches its current behavior after the change. Out of scope: pre-existing name mismatches the diff does not touch.

9. **Compactness over clarity** — nested ternary operators, dense one-liners, expressions that pack multiple conditions / lookups / transformations into a single chain. Prefer `if` / `else` / `switch` / named intermediate values when expanding makes the intent legible. Specific anti-patterns: nested ternaries (`a ? b : c ? d : e` — almost always replaceable with `if`/`else` or `switch`), deeply chained method calls without intermediate names, single-line conditionals that hide branches.

## Balance rails — anti-over-simplification

Apply these as **negative space**: a candidate `mechanical_edit` that would violate any of these is **not actionable** as a mechanical fix even when it matches one of items 1–9. Either downgrade to `structural_note` or skip the finding entirely.

- **Don't sacrifice readability for fewer lines** — if "simpler" means "denser", it is not actually simpler.
- **Don't remove abstractions that are pulling their weight** — a helper that is called from N call sites, or that names a domain concept, is earning its existence even if N is small. Items 1 / 3 target abstractions that show no such use; everything else stays.
- **Don't combine unrelated concerns into one function / component** to collapse line count. Cohesion matters more than line count.
- **Don't replace explicit code with overly clever idioms** — clever shortcuts that require the reader to mentally simulate the language semantics to understand are net negative.
- **Don't make the code harder to debug or extend** — if the cleanup removes a useful stepping stone (a named intermediate value, a deliberately verbose error path), keep it.
- **Don't change observable behavior** — see § Preserve functionality. This rail is reproduced here because over-simplification is the most common path to silent behavior change.

## Overlap handling

When a finding could match more than one item, apply these rules and emit only the preferred classification. **First-match-wins per finding** — never split one finding across multiple items.

- **(1) + (3)** — a single-call-site helper that looks like both duplication and over-abstraction: **prefer (3) over-abstraction**. The defect is that the abstraction was unnecessary, not that the same code repeats.
- **(2) + (5)** — code added "for future use" but never called within the diff: **prefer (5) speculative-features** when the intent reads as "future use" in adjacent prose / comments; **prefer (2) dead code** when no such intent is signaled and the code simply has no caller.
- **(4) + (5)** — a try/catch / null-guard around an operation that currently cannot fail, added "for a future path": **prefer (5) speculative-features**.
- **(6) + (7)** — prose redundancy in SKILL.md / `references/*.md`: **prefer (7) redundant prose** by default; **prefer (6) comment narration** only when the redundant content has the line-by-line paraphrase structure of narration / preamble (not just general repetition).
- **(8)** is independent — rename propagation gaps are a **consistency** layer, not a duplication layer. Apply (8) on its own; do not collapse it into (1).
- **(9) + (6)** — a dense one-liner whose intent would become clear if the obvious-`what` comment were instead reflected in the code structure: **prefer (9) compactness** — fix the code shape, then the comment falls out of (6) automatically. Apply (6) on its own only when the code shape is already clear.

## Reviewer judgment notes

- The checklist is a starting frame, not a constraint. If a real cleanup opportunity does not fit any item cleanly, surface it as a `structural_note` with the reason — the caller can route it to a human.
- Project conventions under `.claude/rules/` and `CLAUDE.md` override this checklist where they conflict. Language-specific or framework-specific standards (import style, function-declaration form, return-type annotation conventions, component / module structure, error-handling patterns) live in those project files — defer to them rather than inferring a standard from the diff.
- Don't chase aesthetic preferences — fix concrete cleanup wins, leave style-only edits to the formatter.
- When in doubt between `mechanical_edit` and `structural_note`, default to `structural_note`. The Preserve-functionality constraint and the Balance rails make `structural_note` the safe fallback whenever there is any risk of behavior change or over-simplification.
