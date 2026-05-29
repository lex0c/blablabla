---
name:        review-diff
description: Self-review the current diff before commit/PR with concrete check passes — correctness, boundaries, coupling, tests, hygiene.
version:     1
trigger_keywords: [review, code review, self review, before commit, before PR, check my changes]
tools:       [bash, edit]
requires:    [DESIGN_SMELLS, ANTI_PATTERNS_AND_CODE_ENTROPY]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "review this before I commit", "check my changes", "anything wrong with this diff". Operates on the **current diff** (staged, or committed but unpushed) — a self-review gate before a commit or PR.

Not a use case: reviewing a large external PR end-to-end (heavier, multi-file context); a security-only review of a boundary (`harden-input-boundary`, `threat-model-component`); chasing a known bug (`debug-failure`).

## Prerequisites

- A diff exists — `git diff`, `git diff --staged`, or `git diff @{u}`.
- The intent of the change is known (what it is *supposed* to do).

## Procedure

1. **State the intent in one sentence.** What should this diff accomplish? If you cannot, obtain it before reviewing — you cannot judge correctness against an unknown goal.
2. **Read the whole diff once, start to finish.** Comprehension only, no judgment. Note anything you do not understand — unclear code is a finding.
3. **Correctness pass.** For each changed function: edge cases (null/empty/zero/negative/overflow), error and early-return paths, off-by-one, and whether the behavior matches the stated intent.
4. **Boundary pass.** Does the diff touch untrusted input, auth, a query, a shell call, or a path? If yes, run `harden-input-boundary` on those lines.
5. **Coupling/cohesion pass — concrete triggers, not taste.** A finding only counts if it hits one:
   - A new function >~40 lines, or one doing I/O + logic + formatting together → flag for split.
   - One logical edit reaching across >~3 modules → coupling smell.
   - A new parameter threaded through >2 callers just to pass one value → flag for restructure.
   - A duplicated block appearing a 3rd time → flag.
6. **Test pass.** Is the changed behavior covered? A bug fix should ship with a regression test (`add-regression-test`); new logic should ship with a test. No test → an explicit justification or a recorded gap.
7. **Hygiene pass.** Leftover debug prints / commented-out code; secrets, keys, tokens; TODOs with no owner; changes unrelated to the intent (scope creep — should be a separate diff).
8. **Report.** List each finding as `file:line — what — why it matters`. If the diff is clean, say so plainly.

## Verification

- Every changed file was read in full (step 2), not just a summary or the hunk headers.
- Each finding is concrete and actionable: `file:line`, the problem, the reason.
- Lines touching a sink/boundary went through `harden-input-boundary`.
- A "no findings" verdict was reached by running all passes — not by skipping them.

## Anti-cases

- Style nitpicks (naming, whitespace) while logic bugs go unreviewed → triage by severity; a linter handles style.
- "Looks good" without reading the full diff → step 2 is mandatory.
- Manufacturing findings to appear rigorous → a clean diff is a valid outcome; report it as clean.
- Treating coupling/cohesion as taste → use the step 5 triggers; if it does not hit one, it is not a finding.
