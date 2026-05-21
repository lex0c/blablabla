---
name:        git-bisect-regression
description: Pinpoint the commit that introduced a bug with an automated binary search (git bisect run).
version:     1
trigger_keywords: [git, bisect, regression, when did this break, culprit commit]
tools:       [bash]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "when did this break", "find the commit that introduced this bug", "this worked last release and is broken now". Use when a bug is **reproducible** and you have a known-good and known-bad point in history.

Not a use case: a bug that was always present (nothing to bisect); a non-deterministic/flaky failure (bisect needs a reliable verdict per commit — see `triage-flaky-test` first); a regression in a dependency, not in this repo's commits.

## Prerequisites

- A reliable reproduction: a command, test, or check that exits **0 = good, non-0 = bad**.
- A known-good commit (old) and known-bad commit (usually `HEAD`).
- A clean working tree — bisect checks out commits.

## Procedure

1. **Write a one-shot test** that reproduces the bug and exits with the right code:
   ```bash
   # example: reduce to a single command
   pytest tests/test_foo.py::test_bar -q
   ```
   Verify it exits non-0 on the bad commit and 0 on the good commit *before* bisecting — a wrong-polarity test inverts the whole search.
2. **Start the bisect:**
   ```bash
   git bisect start
   git bisect bad <bad-sha>      # or just: git bisect bad   (defaults to HEAD)
   git bisect good <good-sha>
   ```
3. **Automate the search** — let git drive the binary search:
   ```bash
   git bisect run <your-test-command>
   ```
   Git checks out the midpoint, runs the command, reads the exit code, and narrows down. Use exit code **125** in your script for "cannot test this commit" (skips it) — e.g. when the build is broken for unrelated reasons.
4. **Read the result:** git prints `<sha> is the first bad commit`.
5. **End the bisect** to restore the original `HEAD`:
   ```bash
   git bisect reset
   ```
6. **Inspect the culprit:** `git show <first-bad-sha>` — confirm the diff plausibly explains the bug.

## Verification

- Re-run the repro on the culprit's parent (`<sha>^`): the bug is **absent**.
- Re-run on the culprit itself: the bug is **present**.
- The culprit's diff has a causal connection to the symptom — if it looks unrelated, the test polarity or a skipped (125) commit may have misled the search; re-run.

## Anti-cases

- Flaky repro → bisect will converge on a random commit; stabilize the test first.
- Build breaks on intermediate commits for unrelated reasons → return 125 from the script to skip, don't return bad.
- Forgetting `git bisect reset` → leaves the repo on a detached midpoint commit; always reset when done.
