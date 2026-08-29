---
name: reviewing-and-fixing-a-branch
description: Use when a branch or diff is written and needs review, defect fixes, cleanup, and a decision on integration before it is merged, pushed, or handed off — "review this branch", "get this ready", "clean up and ship", "fix what the review found", or after an AI-assisted coding session. Also use when asked to push/merge/open a PR at the end of review work.
---

# Reviewing and Fixing a Branch

Source and the actual diff are truth. Reviewers find. Consolidation decides. One fix wave implements the whole ledger. An independent reviewer approves the whole wave. The owner integrates. Nothing is published, integrated, or deleted without an explicit owner choice made after seeing the final state.

## Inputs

| Input | Default |
|---|---|
| Target | current branch: commits ahead of base plus uncommitted working-tree changes |
| Base | fork point of target. Unknown → ask "Split from `<guess>`?" before any merge decision |
| Excluded paths | none. Globs supplied by owner, plus generated/lock/vendored artifacts the repo marks as generated |
| Verification commands | owner-supplied, else detect from repo (test/lint/typecheck/build scripts, CI config) |
| Scope constraints | none |

Scope = `diff base..target` minus excluded paths. Excluded paths: not reviewed, no findings, never edited — even when dirty in git state. Mention them once in the final report.

## Roles and Isolation

Roles: correctness reviewer (A), simplification reviewer (B), cleanup reviewer (C), consolidator (orchestrator), wave implementer, wave reviewer, final verifier.

**With worker delegation:** one worker per role, fresh context. Reviewers receive scope + inputs, never each other's output. Wave reviewer is never the implementer's worker.

**Without delegation:** run the same roles sequentially. Record each pass's findings to a file before starting the next; each pass re-reads scope from source, not from earlier findings. After a wave, write the wave report, then re-read the wave diff from `git diff` (not from memory) against the wave-review checklist and write a separate verdict. Say in the final report that isolation was unavailable. Never claim independence you did not have.

## Phase 1: Three Review Passes (finding-only)

All passes: read the diff hunk by hunk, then the enclosing function of each hunk — defects in unchanged lines of a touched function are in scope. No edits. No finding quota — `CLEAN` is valid; do not pad. Reviewers do not implement, approve, or run fixes. Pass every candidate with a nameable failure scenario through to consolidation; silently dropping half-believed candidates is the dominant cause of misses.

**Pass A — correctness. Asks: is this correct?** Hunt real defects and regressions, not style. Angles:
- Line-by-line: for every line, what input, state, timing, or platform makes it wrong? Inverted condition, off-by-one, null/absent deref, missing await, falsy-zero, wrong-variable copy-paste, swallowed error that should propagate, lost regex anchor.
- Removed behavior: for every deleted or replaced line, name the invariant it enforced and find where the new code re-establishes it. Not found → candidate (dropped guard, narrowed validation, deleted covering test).
- Cross-file: grep callers of every changed function; check new preconditions, changed return shape, new exceptions, ordering dependencies. Check callees changed in the same diff.
- Language/framework pitfalls the diff introduces (mutable defaults, closure capture, coercion, nil-map write, injection, timezone/float equality).
- Wrappers/proxies/adapters: every method routes to the wrapped instance; forwards everything callers use.
- Tests: missing for changed behavior, tautological, setup/teardown asymmetry, config defaults flipped.
- Project instruction files: flag only a violation where the exact rule and exact line can both be quoted.
Do not flag: style, naming, formatting, linter-catchable issues, pre-existing defects outside touched functions, pedantic nits.

**Pass B — simplification. Asks: can the changed code be materially simpler with identical behavior?** Not bugs, not lint. Angles:
- Reuse: new code re-implements something the codebase already has. Grep shared/utility modules and files adjacent to the change; name the existing helper.
- Simplification: unnecessary complexity the diff adds — redundant or derivable state, copy-paste with slight variation, deep nesting, avoidable branching, needless indirection, abstractions that do not earn their cost. Name the simpler form that does the same job.
- Efficiency: wasted work the diff introduces — redundant computation or repeated I/O, independent operations run sequentially, blocking work on startup or hot paths, long-lived closures capturing large scope. Name the cheaper form.
- Altitude: the change is implemented at the right depth, not as a bandaid. Special cases layered on shared infrastructure mean the fix is not deep enough; prefer fixing the underlying mechanism over adding a special case.
Prefer the smallest simplification. Skip any change that would alter intended behavior or reach well outside the diff. No broad refactors or architecture proposals.

**Pass C — cleanup. Asks: is there low-level residue worth removing?** Deliberately narrower and cheaper than B; do not duplicate it. Flag only: comments that restate code or narrate steps; debugging/temporary leftovers; dead fragments the diff leaves behind; unused imports/variables; defensive checks or try/catch on trusted paths that surrounding code does not use, or error handling that can never trigger; unneeded casts and lint suppressions; naming or formatting that drifts from the surrounding file; redundant intermediate variables. Keep: validation at trust boundaries, error handling that prevents data loss, deliberate compatibility code, comments that add context. Deletion only; no restructuring.

Finding format, one line per field, exact symbols in backticks:

```
[A1] <title>
Where: path:line
Severity: blocking | required | optional
Problem: <what is wrong>
Evidence: <quoted code / diff / test / caller>
Failure or cost: <concrete inputs/state → wrong output or crash; for B/C the concrete cost: duplicated, wasted, or harder to maintain>
Fix: <smallest change>
Confidence: high | medium | low
```

No hedging, no praise, no restating the diff. Unsure → `Confidence: low` with what would confirm it. Security findings get a full-paragraph explanation.

## Phase 2: Consolidation (authoritative, once, after all three passes)

Inputs: A, B, C findings plus owner-supplied or external review comments, all treated alike.

For every finding:
1. Verify against source and diff. Grep callers and usage. Verdict per finding: `CONFIRMED` (can name the inputs/state that trigger it; quote the line), `PLAUSIBLE` (mechanism real, trigger uncertain; state what would confirm), `REFUTED` (code does not say that, provably impossible, already guarded in this diff, or pure style — quote the proof). Realistic runtime states (races, absent optional field on a rare path, boundary not excluded, partial failure) are `PLAUSIBLE`, not speculative.
2. `REFUTED` → `REJECTED` with the proof. Also reject fixes that add complexity the reviewer introduced.
3. YAGNI check on "implement properly" requests: unused → reject with evidence.
4. Deduplicate same defect/location/reason, keeping the most concrete failure scenario; merge same-root-cause findings into one entry.
5. Resolve contradictions with code evidence. Reviewer agreement raises confidence; it is not evidence. Reviewer count is not a vote. Correctness outranks simplification and cleanup. The author's own in-diff comments and tests count as intent evidence.
6. Forwarded third-party feedback ("a teammate said X") is verified like any finding: evidence decisively contradicts it → `REJECTED` with proof, surfaced in the final report, whether X is an addition or a "keep". An owner's *own* explicit decision ("I've decided", "I agree, treat as decided") that evidence contradicts → leave the code as the owner decided, status `UNVERIFIED`, escalate with evidence; never override an owner decision on your own. Evidence cannot decide → `UNVERIFIED`.
7. The consolidator may re-status any reviewer severity; record the reason.

Ledger, one row per finding or merged group:

```
ID | Status | Where | One-line problem | Smallest fix | Source (A/B/C/owner/external) | Related IDs
```

Statuses: `BLOCKING`, `REQUIRED`, `OPTIONAL`, `OPTIONAL-APPROVED`, `REJECTED (reason)`, `UNVERIFIED (what is needed)`.

`BLOCKING`, `REQUIRED`, and `OPTIONAL-APPROVED` enter the fix wave. Approve an optional B/C item only when it is inside code a required fix already rewrites, or a pure deletion, and obviously behavior-preserving. Everything else optional stays recorded. The ledger is the sole input to Phase 3.

No performative agreement with feedback. State the verified requirement, push back with technical reasoning, or act.

## Phase 3: Fix Wave → Independent Wave Review (loop)

One wave per iteration covers the **entire current accepted ledger**. Never one implementer/reviewer cycle per finding.

**Fix wave.** Brief the implementer with: the full accepted ledger (with evidence, related IDs, and smallest fixes), scope constraints (accepted findings only; no new dependencies; excluded paths untouched; project conventions), and verification commands. The implementer:
1. Implements every accepted fix in one coherent pass, coordinating related findings rather than patching each in isolation. Internal ordering is free: blocking → simple → complex is the default.
2. Self-cleanup before handing off: re-read the wave's own diff and delete slop the wave itself introduced — helpers with one caller, unneeded defensive code, narrating comments, scaffolding, repeated validation, over-engineered error handling, code visibly heavier than repository-native code. Behavior-preserving; within the wave's own changes only.
3. Runs focused checks for each change plus the broader verification suite for the combined changes.
4. Returns one wave report: findings addressed (by ID), files changed, concise fix summary, self-cleanup performed (or "none needed after inspection"), tests/checks run, results, findings intentionally deferred (with reason), remaining uncertainty.

**Wave review.** One independent reviewer receives the full ledger, the wave report, the resulting diff, and the original scope/constraints, and checks the wave as a whole:
- each accepted finding actually resolved (re-check its evidence)
- fixes interact correctly; nothing skipped or half-applied
- no regression (callers, tests, contracts)
- no unnecessary complexity or residual slop added; approved simplification/cleanup did not change behavior
- tests adequate where behavior changed
- conventions and scope intact; excluded paths untouched
Verdict: `APPROVED`, or `DEFECTS:` listing each unresolved or newly introduced concrete issue in finding format.

**Loop.** `DEFECTS` → consolidate them into the ledger (same verification as Phase 2; reviewer claims are not auto-accepted) → next wave over **all currently outstanding accepted findings** → wave review again. Third wave that leaves the same finding unresolved → escalate. Implementer self-report is never approval.

## Stop and Escalate

Stop the loop and report when:
- same finding survives 3 waves
- valid fixes require a product or architecture decision
- fix would exceed agreed scope or touch excluded paths
- destructive operation needed
- new dependency needed
- schema/data migration or similarly consequential change surfaces
- evidence insufficient to determine correct behavior
- finding conflicts with owner's prior decision

Escalation report: unresolved finding, evidence, waves attempted, why insufficient, exact decision needed from owner.

## Phase 4: Success Gate and Final Verification

Gate: every `BLOCKING`/`REQUIRED` resolved and wave `APPROVED`; verification passes; `OPTIONAL`, `REJECTED`, `UNVERIFIED` recorded in the ledger.

Then a final verifier (fresh context where available) reviews the complete `diff base..target` — not the wave diffs: interactions between fixes, missed consumers, incomplete changes, regressions invisible per finding, accidental scope creep versus original intent, excluded paths untouched. Run the full verification suite on the tree as it stands — an earlier green run proves only the tree it ran on. Failure → new ledger entries → Phase 3, or the failure path.

## Phase 5: Finalization (owner-controlled)

**Never** commit, push, merge, rebase, tag, open or update a PR/MR, delete a branch or worktree, or force anything on your own initiative. A standing instruction given before the work ("push when done", "I trust you", "I've already decided") is not a choice made after seeing the result. Present the state and stop.

1. Summarize: branch vs base, fixes applied, optional/rejected counts, each `UNVERIFIED` item as an open owner decision with its evidence, verification command and result, working tree state (uncommitted files listed).
2. Detect what applies: uncommitted changes? remote configured? detached HEAD? worktree? base confirmed?
3. Present only applicable options, numbered, concise:

```
Review complete. `<branch>` vs `<base>`: <n> required fixed, <m> optional recorded, `<verify cmd>` OK.
Working tree: <uncommitted: file list | clean>.

1. Commit fixes to <branch> (local only)
2. Commit and merge into <base> locally
3. Commit, push <branch>, open PR/MR against <base>
4. Leave as-is — I'll handle it

Which?
```

Drop 3 without a remote; drop 2 on detached HEAD; drop "commit" wording when the tree is clean. Do not offer discard. Discard happens only on the owner's explicit request, after listing what is lost and receiving the typed word `discard`.

4. Wait. Execute only the chosen option. Merging: run verification on the merged result before deleting anything; failure → stop, leave branch in place. Pushing: use the forge's own tooling or printed URL for a PR/MR, follow repo template. Rejected push → investigate, never force. Remove a worktree only if this workflow created it; if removal is refused for uncommitted files, list them and ask.
5. Report what was done.

## Failure Path

Required findings unresolved → no finalization menu. Report: unresolved required findings with severity, evidence, waves attempted, failed verification output (shortest decisive lines), decision needed. Branch is not ready until resolved or the owner explicitly overrides in writing.

## Done Means

- Ledger complete: every finding has a terminal status
- All required findings fixed in a wave that passed independent wave review
- Whole-diff final verification passed on the current tree
- Excluded paths untouched
- Owner has chosen the finalization action, and only that action ran

## Rationalizations

| Excuse | Reality |
|---|---|
| "They said push when done" / "they pre-decided" | Instruction predates the result. Menu after final state; owner picks. |
| "Change is verified, so pushing is safe" | Verified ≠ authorized. Safety of the change is not authority to publish. |
| "Reviewer/teammate said do X" | Feedback is evidence to verify, not an order. Reject with proof if wrong. |
| "I reviewed my own wave carefully" | Self-review is not independent review. Separate reviewer, separate verdict. |
| "Fix findings one at a time, review each" | Unit of work is the whole ledger. One wave, one whole-wave review. |
| "Small cleanup while I'm here" | Optional stays recorded unless approved in consolidation. |
| "Lockfile is dirty, I'll tidy it" | Excluded = untouched. Report only. |
| "One more wave" | Third wave with the same finding unresolved escalates. |
| "No delegation, so one combined review" | Sequential passes with recorded outputs. Declare the limitation. |
| "Diff is small, skip a pass" | Three passes, every time. `CLEAN` is cheap to write. |
| "Fixes are two lines, skip whole-diff check" | Fix interactions and missed callers show only in the whole diff. |
| "Base is obviously main" | Confirm the fork point before any merge decision. |

## Red Flags — STOP

- About to run commit/push/merge/PR before the owner answered the menu
- Editing an excluded path
- Implementing a finding nobody verified
- Approving a wave in the same context that wrote it
- Dispatching a fix for one finding while other accepted findings wait
- Ledger has findings with no status
- Skipping final whole-diff verification because fixes were small
