---
name:        add-regression-test
description: Capture a known bug as a test that fails before the fix and passes after — proving the bug and guarding it.
version:     1
trigger_keywords: [regression test, reproduce bug, test for bug, prevent regression, cover the fix]
tools:       [bash, edit]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "write a test for this bug", "make sure this does not regress", "cover the fix". Use when a **specific, known bug** must be pinned by a test — typically alongside or just before fixing it.

Not a use case: broad coverage of untested code (that is `increase-coverage`); pinning the current behavior of legacy code before refactoring (that is `add-characterization-tests` — there the behavior is treated as correct; here it is treated as a bug).

## Prerequisites

- A clear, reproducible description of the bug: input, expected output, actual (wrong) output.
- Knowledge of where the test belongs and how the suite is run.

## Procedure

1. **Pin down the minimal trigger.** Reduce the bug to the smallest input/state that produces the wrong behavior — that is the test case. If a fix is not written yet, this overlaps with `debug-failure` step 2.
2. **Write the test asserting the *correct* behavior** — what *should* happen, not the current buggy output. Place it with the related tests; name it so it reads as the bug ("test_negative_quantity_rejected", and reference the issue ID in a comment).
3. **Run it against the unfixed code — it must FAIL.** This is the load-bearing step: a regression test that has never been seen failing proves nothing. If it passes now, it does not exercise the bug — go back to step 1.
4. **Apply the fix** (or, if the fix already exists, temporarily revert it to do step 3).
5. **Run the test again — it must now PASS.**
6. **Run the full suite** — confirm the new test and fix broke nothing else.

## Verification

- The test **failed on the unfixed code and passes on the fixed code** — both halves observed, not assumed.
- The failure message (when it fails) is legible — it points at the bug, not just "assertion error".
- The test is deterministic — run it 10× isolated; if it flakes, fix that now (see `triage-flaky-test`).
- The full suite is green.

## Anti-cases

- Writing the test only after the fix and never seeing it fail → it may assert the wrong thing or not reach the bug; step 3 is mandatory.
- Asserting the buggy output to "make it pass" → that locks the bug in; assert the correct behavior.
- A test so broad it would fail for many unrelated reasons → scope it to *this* bug so a future failure is diagnostic.
