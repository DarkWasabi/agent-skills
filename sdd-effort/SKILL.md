---
name: sdd-effort
description: Use when dispatching implementer or reviewer subagents in subagent-driven-development (or any skill that fans out plan tasks to subagents) and the tasks vary in how much exploration, judgment, or verification each one deserves.
---

# SDD Effort Classification

## Overview

Model choice controls *capability* (which model runs the subagent). Effort
is a separate axis: how much exploring, verifying, and self-checking that
subagent should do before reporting back. Nothing in the Agent tool sets
effort for a subagent — the only lever is what you write into its dispatch
prompt. Left unstated, subagents default to a uniform middle ground
regardless of whether the task is a two-line rename or an open-ended
architecture decision.

**Core principle:** classify each task's required effort, then put that
classification in the dispatch prompt as literal text — not just in your
own reasoning about which model to pick.

## When to Use

Alongside superpowers:subagent-driven-development (or executing-plans),
right where you already compose the dispatch prompt — same moment you're
deciding the task's model tier, before you send it.

Skip it for a plan where every task is genuinely uniform in shape — nothing
to classify.

## Classifying a Task

Read the task's brief and score it against these signals:

| Signal | Low effort | Medium effort | High effort |
|---|---|---|---|
| Spec completeness | Exact values/steps given | Intent given, some cases unstated | Open-ended, no enumerated behavior |
| File/interface breadth | 1–2 known files | Several files, known interfaces | Unknown scope, new interfaces to design |
| Precedent | N/A — mechanical | Existing pattern to extend | No existing pattern in the codebase |
| Cost of a wrong call | Trivial to fix | Moderate — a fix round catches it | Expensive to unwind (storage/schema/API shape) |

**Any single "high" signal wins** — one architectural or expensive-to-unwind
element makes the whole task high effort even if the rest reads mechanical.
Otherwise: all-low → low; anything else → medium.

This is the same lens subagent-driven-development's Model Selection section
uses for model tier — reuse your classification, don't re-derive it twice.

## The Dispatch Line

Add one line to the dispatch prompt, its own paragraph, labeled `Effort:` —
structural placement means you (and any reviewer of your dispatch) can spot
whether it's missing at a glance. Use the phrasing verbatim; don't paraphrase
into vaguer language ("try to be careful") that gives the subagent room to
negotiate:

**Low:**
> Effort: low — quick, mechanical pass. The spec is complete; implement
> directly, minimal exploration beyond the brief, single verification pass
> before reporting.

**Medium:**
> Effort: medium — standard implementation. Read the touched interfaces,
> verify each edge case against the brief, run the full covering suite
> before reporting.

**High:**
> Effort: high — work thoroughly. Explore existing patterns before
> designing anything new, consider edge cases and failure modes explicitly,
> self-review against the global constraints before reporting, and stop for
> NEEDS_CONTEXT rather than guess on any expensive-to-unwind decision.

Reviewer dispatches (task-reviewer-prompt, re-review-prompt) get the same
line, scaled to the diff's size and risk rather than the task's — a large
mechanical diff can still be low-effort review; a three-line change to
auth logic is high-effort review regardless of diff size.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Effort decided in your head, never written into the prompt | The subagent only sees the dispatch text. If the line isn't in the prompt, the classification didn't happen. |
| Conflating effort with model tier | State both independently. A cheap model can carry "medium effort" on a fiddly-but-narrow task; a capable model can run "low effort" on a big transcription job to keep it terse. |
| Vague thoroughness language ("be careful", "do a good job") | Use the three phrasings verbatim — vague language is negotiable, the recipe isn't. |
| Same effort line copy-pasted across every task in the plan | Re-classify per task. A plan with one architectural task and four mechanical ones should read that way in the dispatches. |

<!-- update-mechanism verification test, safe to ignore -->
