---
name: spec-review
description: Review a completed design spec against the current codebase before implementation planning. Use independent reviewers to find inconsistencies, hidden risks, and unnecessary complexity, then obtain owner decisions and apply only approved changes to the spec.
---

# Spec Review

Review a completed design spec after brainstorming and before `writing-plans`.

Goal: make spec implementation-ready without redesigning it unnecessarily.

## Rules

- Review only. Do not implement feature or write implementation plan.
- Treat spec claims as hypotheses, not evidence.
- Verify important claims against current code, config, tests, schemas, and established repository patterns.
- Prefer current source code over generated docs, indexes, summaries, or stale architecture descriptions.
- Do not manufacture findings. A clean review is valid.
- Ignore wording or style preferences unless ambiguity could change implementation.
- Prefer smallest correction that resolves finding.
- Do not introduce speculative flexibility, abstractions, extension points, or future-proofing.
- Reviewers MUST remain read-only.
- Only orchestrator may edit spec.
- Spec changes require explicit owner approval.
- Preserve existing spec purpose, structure, scope, approved decisions, and file location unless owner explicitly changes them.
- Keep spec as design document. Do not turn it into implementation plan.

## Workflow

### 1. Establish Review Context

Read complete spec.

Inspect only codebase areas needed to validate design, including where relevant:

- existing implementations and patterns
- APIs and internal contracts
- types and schemas
- state and data boundaries
- integration points
- lifecycle behavior
- relevant tests
- relevant configuration

Build concise context for reviewers.

Do not perform full-repository exploration unless evidence shows it is needed.

### 2. Run Independent Reviewers

Use independent subagents or workers when runtime supports them.

Run up to 3 reviewers in parallel.

Each reviewer:

- receives complete spec
- receives relevant repository context
- may inspect additional relevant source
- works independently
- MUST NOT see other reviewers' findings
- MUST NOT edit spec or code

Use distinct review angles.

#### Reviewer A: Codebase Reality

Check whether design agrees with actual codebase.

Focus on:

- incorrect assumptions about current behavior
- incompatible APIs, types, schemas, state, or lifecycle
- contradictions with existing architecture
- spec sections that contradict each other
- existing mechanisms the spec unnecessarily duplicates
- integration effects omitted from spec
- affected modules or contracts the design overlooks

#### Reviewer B: Hidden Risks

Find implementation-affecting gaps.

Focus on:

- missing states or transitions
- missing error and recovery behavior
- edge cases
- ordering assumptions
- lifecycle assumptions
- async or concurrency behavior where relevant
- backward compatibility
- migration effects
- cleanup or ownership behavior
- regression risks
- requirements that cannot be tested deterministically

Only report issues material to design or later implementation.

#### Reviewer C: Simplicity

Challenge unnecessary complexity.

Focus on:

- YAGNI
- premature abstractions
- unnecessary layers or indirection
- duplicated concepts
- speculative configurability
- generalized solutions for one concrete requirement
- new mechanisms where existing repository patterns suffice
- simpler designs that preserve required behavior

Do not propose broad architectural rewrites unless current design has a material problem.

### 3. Reviewer Output

Every finding MUST use this format:

```text
Finding: <short title>
Spec: <section or claim>
Evidence: <file, symbol, test, config, or other concrete evidence>
Issue: <specific problem>
Impact: <what could fail or become ambiguous during planning or implementation>
Direction: <smallest reasonable correction>
Confidence: high | medium | low
```

Reviewer requirements:

- verify load-bearing assumptions rather than trusting spec
- distinguish verified facts from inference
- provide concrete evidence for codebase claims
- report only actionable findings
- do not create a finding quota
- return no finding when evidence does not support one

If no material issues exist:

```text
CLEAN
```

If independent subagents are unavailable, report that independent review could not be performed.

Do not simulate multiple independent reviewers inside one shared reasoning context while claiming independence.

### 4. Consolidate Findings

After reviewers finish, orchestrator independently evaluates their reports.

For each finding:

1. Deduplicate findings with same root cause.
2. Verify supporting evidence when practical.
3. Reject unsupported or speculative claims.
4. Reject style-only findings.
5. Reject complexity introduced by reviewer itself.
6. Resolve factual disagreements using current codebase evidence.
7. Keep only findings worth owner consideration.

Reviewer findings are advisory.

Reviewer count is not a vote.

Agreement between independent reviewers increases confidence but does not replace evidence.

### 5. Owner Decision Gate

Do not modify spec yet.

Present retained findings to owner.

Group findings when they share one design decision.

For each owner decision, use:

```text
[F1] <finding title>

Problem:
<concise explanation>

Evidence:
<concise spec/code evidence>

Recommendation:
<Option N> — <reason>

Choices:
- Discard — keep current spec unchanged
- Option 1 — <smallest viable correction>
- Option 2 — <meaningfully different viable correction>
- Option 3 — <meaningfully different viable correction>
```

Rules:

- Always include `Discard`.
- Provide only viable alternatives.
- Up to 3 change options.
- Do not invent weak options merely to reach 3.
- Make recommendation explicit.
- Keep owner decision surface concise.
- Do not edit spec before owner responds.

### 6. Apply Owner Decisions

After owner provides decisions:

- apply only explicitly approved changes
- leave discarded findings unchanged
- treat selected option as decision authority
- make smallest spec edit that fully records decision
- update dependent sections when required for consistency
- remove text made obsolete by approved decision
- do not introduce unrelated improvements

Preserve Superpowers design-spec contract:

- keep existing spec file and location
- preserve design-document role
- preserve approved decisions unless owner supersedes them
- preserve scope and out-of-scope boundaries unless owner changes them
- keep requirements explicit
- keep design internally consistent
- leave no new `TODO`, `TBD`, placeholder, or unresolved ambiguity
- do not add implementation task breakdowns
- do not invoke or reproduce `writing-plans`

### 7. Integrity Check

After editing spec, inspect changed sections and their direct dependents.

Verify:

- every change maps to an owner-approved decision
- no discarded finding was applied
- approved decisions are represented completely
- no contradiction was introduced
- terminology remains consistent
- architecture remains coherent
- scope did not expand accidentally
- spec remains suitable input for `writing-plans`

Do not launch another full review wave unless approved changes materially invalidate assumptions used by previous reviewers.

## Completion

If all owner decisions are resolved:

```text
READY_FOR_WRITING_PLANS

Applied:
- <approved decision>
- <approved decision>

Discarded:
- <discarded finding>
```

If owner decisions remain unresolved:

```text
OWNER_DECISIONS_REQUIRED

Pending:
- <finding ID>
- <finding ID>
```

Do not proceed to `writing-plans` from this skill.
