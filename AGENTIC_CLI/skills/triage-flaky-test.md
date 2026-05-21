---
name:        triage-flaky-test
description: Diagnose a non-deterministic test — confirm flakiness, isolate the cause, fix it or quarantine with a reason.
version:     1
trigger_keywords: [flaky test, intermittent, non-deterministic, test fails sometimes, heisenbug, ci flake]
tools:       [bash, edit]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "this test is flaky", "the test fails sometimes in CI", "passes locally, fails on the runner". Use when a test gives **different verdicts on the same code**.

Not a use case: a test that fails deterministically (that is a real bug or a real regression — use `debug-failure`); a test that has never passed (it was written wrong).

## Prerequisites

- The ability to run the test repeatedly and in isolation.
- The failing output from at least one flaky run (CI log, stack trace).

## Procedure

1. **Confirm it is flaky.** Run it many times, unchanged:
   ```bash
   for i in $(seq 1 50); do <run-the-single-test> || echo "FAIL run $i"; done
   ```
   If it never fails here, the flake is environment-dependent (CI parallelism, load, a different machine) — reproduce under those conditions.
2. **Isolate the cause.** Flakiness almost always comes from one of these — test each hypothesis:
   - **Order dependence / shared state:** run the test alone vs. in the suite; run the suite in randomized order. If the verdict changes, a previous test leaks state (global, DB row, file, singleton).
   - **Timing / races:** `sleep`-based waits, real timers, async without proper await, concurrency. Look for fixed sleeps and unsynchronized assertions.
   - **Unseeded randomness:** random data, random IDs, hash/set iteration order. Re-run with a fixed seed — if it stabilizes, that is it.
   - **Time / locale / timezone:** tests that depend on `now()`, date boundaries, locale-sensitive formatting.
   - **External dependency:** real network, real clock, a shared service — anything not hermetic.
3. **Fix the root cause:** isolate state (fresh fixtures, teardown), replace sleeps with explicit waits/synchronization, seed randomness, inject a fixed clock, stub external calls.
4. **If it cannot be fixed now**, quarantine explicitly — mark it skipped/`xfail` with a comment stating the reason and a tracking reference. A silent skip is worse than a flake.

## Verification

- The loop from step 1 (50+ runs) is now **all green**.
- Run it in randomized suite order and in isolation — same verdict both ways.
- The fix names a concrete mechanism (e.g. "shared DB row", "unseeded RNG") — "ran it again and it passed" is not a diagnosis.
- If quarantined: the skip carries a written reason and a ticket, not a bare annotation.

## Anti-cases

- "Re-run until green" as the fix → that hides the flake, does not remove it; it will fail in CI again.
- Adding a longer `sleep` → trades a fast flake for a slow flake; replace the sleep with a real wait condition.
- Deterministic failure mislabeled as flaky → if it fails every time, it is a bug; hand off to `debug-failure`.
