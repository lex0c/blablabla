---
name:        debug-failure
description: Find the root cause of a reproducible failure by measurement — reproduce, minimize, hypothesize, verify.
version:     1
trigger_keywords: [debug, bug, crash, failing test, wrong output, root cause, broken]
tools:       [bash, edit]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "this is broken, find out why", "debug this crash / failing test / wrong output". Use when something behaves wrong and you need the **root cause**, not a guess.

Distinct from siblings: `git-bisect-regression` finds *which commit* introduced it; `profile-hotspot` finds *where time goes*. This skill finds *why the behavior is wrong*. If the bug is a recent regression, run `git-bisect-regression` first — the culprit commit is a huge head start.

Not a use case: a feature that was never implemented (that's building, not debugging); a flaky/non-deterministic failure — stabilize the repro first or it cannot be debugged reliably.

## Prerequisites

- A clear statement of expected vs. actual behavior.
- A way to run the failing scenario.

## Procedure

1. **Reproduce reliably.** Get the failure to happen on demand — a command, a test, a script. If you cannot reproduce it, you cannot debug it; first invest in a stable repro. Note the exact symptom (error message, wrong value, exit code).
2. **Minimize the repro.** Strip the scenario down: remove inputs, code paths, config until you have the smallest case that still fails. Each thing you remove that *keeps* the failure is a thing the bug does not depend on — that is signal, not wasted effort.
3. **Locate, by bisection.** Narrow *where* the wrong behavior originates: binary-search the data flow — log/inspect a value at the midpoint between "known correct" and "known wrong", then recurse into the half that is wrong. Do not read the whole codebase; halve the search space each step.
4. **Form one hypothesis.** State a specific, falsifiable cause ("the value is wrong because X is null here"). One at a time.
5. **Instrument to test the hypothesis.** Add a targeted log/assert/breakpoint that will *prove or disprove* it. Run. If disproved, discard it and return to step 4 — do not pile guesses.
6. **Fix the root cause**, not the symptom. A fix that hides the symptom (catch-and-ignore, clamp the value) without explaining the mechanism is not a fix.
7. **Verify** (see below).

## Verification

- The minimized repro from step 2 now **passes**.
- You can explain the *mechanism* — why the bug happened and why the fix addresses it. If you cannot, you patched a symptom; keep going.
- Add a regression test that fails without the fix and passes with it.
- The full test suite still passes — the fix introduced no new breakage.

## Anti-cases

- Trying fixes blindly without a hypothesis ("change this and see") → brute force, not debugging; if a change works but you cannot say why, you do not have the root cause.
- Debugging local code when the failure is in a dependency, the DB, or the network → check end-to-end before diving in.
- "Fixing" by suppressing the error → the bug is still there; step 6 exists to prevent this.
