---
name: clean-code-refactor
description: Use when restructuring existing code for readability, structure, or testability without changing what it does — a function or class has grown long, duplicated, or tangled, or someone asks to "clean up", "tidy", "refactor", "modernize", "improve", or make code "production-grade". Not for adding features, fixing bugs, or changing APIs. Stack-agnostic.
---

# Clean Code Refactor

## Overview

A refactor changes code **structure**, never its **observable behavior**. Same inputs → same outputs, same side effects, same errors, same public surface. If a change alters any of those, it is not a refactor — it is a behavior change wearing a refactor's clothes.

**Violating the letter of behavior-preservation is violating the spirit of refactoring.**

The trap is framing: *"make it production-grade"*, *"modernize this"*, *"the reference example for the team"* read as permission to redesign. They are not. A refactor with no behavior change is the whole job.

## When to use

- A function/method is long, or a class carries multiple responsibilities.
- Duplicated logic across call sites; magic values; unclear names; tangled concerns.
- Someone says "clean up", "tidy", "refactor", "modernize", "improve", "make it production-grade / maintainable".

## When NOT to use (these are separate tasks)

- **Fixing a bug** — that changes behavior. Refactor is behavior-preserving.
- **Adding a feature** or changing an API/contract/schema.
- Found a bug *while* refactoring? **Surface it, don't silently fix it.** Note it, finish the refactor, fix it as its own change.

## The discipline

1. **Preserve behavior exactly.** No "while I'm here" fixes. No making inconsistent things consistent (adding the retry the other two functions have *is a behavior change*). No "better" defaults. An obvious bug or an obviously-better behavior is still out of scope — surface it separately.
2. **Add no abstraction the code didn't already have.** No new classes, factories, dependency injection, interfaces, strategy patterns, or config layers unless the codebase already uses them *and* the task asked for them. "Production-grade" and "reference example" are not licenses to redesign. YAGNI.
3. **Algebraically equal ≠ identical.** `price - price*d` → `price*(1-d)`, `=== 200` → `2xx`, two `Date.now()` calls → one shared value — each is a behavior change (floating-point, edge cases, side-effect timing). Keep the exact expression, comparison, and call sequence.
4. **Don't touch the public surface.** Signatures, parameter order, exports, return shapes, error codes stay. Grep callers *before* changing any of them; if a change is unavoidable, update every caller in the same commit.
5. **Match the surrounding style.** Don't blanket-add strict modes, `final`, type annotations, or a new file layout the code doesn't use.
6. **Slice small; validate each slice.** One concern per edit. Run the project's own format / lint / type / test commands after *each* slice, never batched at the end.
7. **Fix, don't suppress.** Resolve type/lint errors. Never add an ignore comment, baseline entry, or blanket disable to "finish."
8. **Tests move with the code; read snapshot diffs.** Don't rewrite a snapshot/golden to match new output without confirming the diff is intentional.

## Workflow

1. **Scope** — read the target plus its callers and its tests. Name the smell.
2. **Discover conventions** — how does *this* codebase name things, structure modules, and place logic? Find existing helpers/utilities to extend instead of inventing new ones. Find its format/lint/type/test commands.
3. **Plan slices** — an ordered list of behavior-preserving steps. Prefer extending existing code over new abstractions.
4. **Apply one slice.**
5. **Validate that slice** — format, lint/types, affected tests, then the full suite.
6. **Update tests alongside** — move a method's test with the method; reuse existing fixtures.
7. **Report** — smells found, slices applied, behavior preserved, and any behavior questions *surfaced* (never silently deferred, never silently fixed).

## Refactor vs. redesign — same task, two outcomes

```js
// Given: the odd one out has no retry and checks an exact 200.
function sendEmail(to, subject, body) {
  const res = http.post(url, { from, to, subject, body });
  return res.status === 200;
}
```

❌ **Not a refactor** — adds a retry to "match the others," broadens `200` → `2xx`, adds logging, wraps everything in a `createNotifier` factory. Every line is a behavior or public-surface change.

✅ **Refactor** — extract the shared payload-build + success-check, rename for clarity, keep the single attempt and the exact `=== 200`. If the missing retry looks like a bug, *say so in the report* — don't add it.

## Reference

`references/implementation-playbook.md` has deeper material for this skill: code-smell/SOLID before-after samples in multiple languages, a refactoring-ROI formula, a technical-debt priority decision tree, quality-metric thresholds (complexity, method/class length, coverage), and a full code-quality checklist. Load it when you need a concrete pattern example or a prioritization call — not needed for small mechanical slices.

## Quick reference — smell → fix

| Smell | Behavior-preserving fix |
|---|---|
| Long function, mixed concerns | Extract well-named functions at one abstraction level |
| Large class / multiple responsibilities | Split by responsibility (SRP) — into the shapes this codebase already uses |
| Duplicated logic across call sites | Extract one shared function; or move the method to where the data lives |
| Long parameter list | Introduce a parameter object *only if the codebase does this* |
| Magic values | Named constants / existing enums |
| Repeated query / N+1 | Existing query scope or eager-load; assert with a test |
| Inline validation repeated | Extract to the project's existing validation seam |
| Type/lint error blocking "done" | Fix the type — never suppress |
| Duplicated test setup | Reuse existing fixtures/factories/traits |

## Rationalizations — STOP, these are the failure

| Excuse | Reality |
|---|---|
| "It's the reference example, so a factory / DI is appropriate" | The task is to clean this code, not redesign it. Add no pattern the code didn't have. |
| "I'll make the three functions consistent" | Consistency that changes what a function *does* is a behavior change. Flag it; don't do it. |
| "`price*(1-d)` is the same as `price - price*d`" | Algebraically, not in floating point / edge cases. Keep the exact expression. |
| "Broadening `200` to `2xx` is more correct" | More correct = a behavior change = a separate task. Preserve `=== 200`. |
| "This isn't gold-plating, it's production-grade" | If you're adding capability, it's not a refactor. Structure only. |
| "Adding logging just improves observability" | New side effects change behavior and can break tests. Out of scope. |
| "Exporting this extra helper is harmless / additive" | The public surface changed. Don't, unless asked. |
| "Tests pass, so behavior is preserved" | Passing tests cover tested paths only. For untested/risky areas, tests aren't proof. |
| "I'll validate everything at the end" | Big-bang hides which slice broke it. Validate each slice. |

## Red flags — STOP and reconsider

- About to fix a bug, add a retry, add validation, or add logging "while I'm in here"
- Adding a class / factory / interface / DI / config layer the codebase doesn't already use
- Rewriting an arithmetic or boolean expression into an "equivalent" form
- Changing a signature, export, return shape, or error code without grepping callers
- Adding a lint-ignore, type-suppression, or baseline entry to "finish"
- Rewriting a snapshot/golden without reading the diff
- Saving all validation for the end
- Your change note says "improved", "modernized", "more correct", or "more consistent" → that is behavior, not structure
