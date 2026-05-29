---
name:        reproduce-bug
description: Turn a bug report into a minimal, deterministic repro before debugging — the trigger you can run on demand.
version:     1
trigger_keywords: [reproduce, repro, can't reproduce, intermittent, minimal repro, isolate bug]
tools:       [bash, edit]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "this is broken", "users report X", a bug report with no reliable way to trigger it. Run this **before** `debug-failure` — you cannot debug what you cannot trigger on demand.

Not a use case: a bug that already has a clean deterministic repro (go straight to `debug-failure`); a test that passes and fails nondeterministically (`triage-flaky-test`).

## Prerequisites

- A bug report, however vague.
- Access to an environment where the bug might occur.

## Procedure

1. **Capture the report verbatim.** Exact input, expected result, actual result, environment (version, OS, config, data state), and frequency (always / sometimes / seen once). Do not paraphrase detail away.
2. **Reproduce as reported.** Same environment, same input, same steps. If it triggers → step 4. If not → step 3.
3. **Close the gap.** The bug depends on a factor not yet captured. Vary **one factor at a time**: data state, timing/concurrency, environment/config, the sequence of prior actions, the user/permissions. A bug that "only happens in prod" depends on a prod-specific factor — name it explicitly.
4. **Reduce to minimal.** Strip every input, step, and piece of data not required to trigger it. Stop when removing one more thing makes the bug disappear — what remains is the essential trigger.
5. **Make it deterministic.** The repro must fire every run. If intermittent, pin the nondeterministic source: seed the RNG, control the clock, force thread ordering, fix the data fixture. An intermittent repro is a half-repro.
6. **Lock it.** Write the repro down — exact steps, a script, or a failing test. Hand off to `add-regression-test` (assert the correct behavior) and `debug-failure` (find the cause).

## Verification

- The repro triggers the bug ≥5/5 runs from a clean state — deterministic and observed, not assumed.
- It is minimal: every remaining step and input is load-bearing — removing any one stops the bug.
- Expected vs actual is stated precisely enough to tell pass from fail unambiguously.
- The repro is captured as text/script/test, not "I saw it happen once".

## Anti-cases

- Handing an intermittent repro to `debug-failure` → you will chase ghosts; make it deterministic first.
- "Cannot reproduce" → closing the report → it reproduces for the reporter; step 3 is unfinished — a factor is uncaptured.
- A repro carrying unrelated setup → the noise misleads the debugger; reduce it (step 4).
- Reproducing once and assuming reliability → run it 5×; once is an anecdote.
