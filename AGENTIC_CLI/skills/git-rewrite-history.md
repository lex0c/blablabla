---
name:        git-rewrite-history
description: Clean up branch history (squash, reword, reorder, drop) non-interactively — no rebase -i required.
version:     1
trigger_keywords: [git, rebase, squash, fixup, autosquash, amend, reword, clean history]
tools:       [bash]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "squash these WIP commits", "fix a typo in an earlier commit message", "reorder/drop a commit before opening the PR". Use to tidy a **local, unpushed (or solely-owned) feature branch** before review.

This skill is built for environments where `rebase -i` is unavailable (no interactive editor). Every step uses non-interactive equivalents.

Not a use case: rewriting history that others have already pulled (forces everyone to recover); editing the shared mainline. If in doubt, branch and rewrite the copy.

## Prerequisites

- A clean working tree (`git status` clean) — stash or commit first.
- Know the base: how far back to rewrite (e.g. `origin/main`, or `HEAD~5`).
- A safety net before starting: `git branch backup/<name>` so a bad rewrite is one `reset` away.

## Procedure

1. **Snapshot for safety:** `git branch backup/pre-rewrite`.
2. **Squash a fix into an earlier commit** — the cleanest non-interactive path:
   ```bash
   git commit --fixup=<target-sha>          # stage the fix, commit as a fixup
   GIT_SEQUENCE_EDITOR=true git rebase --autosquash -i <base>
   ```
   `GIT_SEQUENCE_EDITOR=true` accepts the generated todo list unedited; `--autosquash` has already moved the `fixup!` commit next to its target. No editor opens.
3. **Reword the latest commit message:** `git commit --amend -m "new message"`.
   Reword an **older** commit: `git rebase --onto` is awkward — instead `git rebase <base>` with `--exec` or, simplest, reset to the commit, amend, and replay:
   ```bash
   git rebase <base> --exec 'true'   # no-op replay to confirm base is clean
   ```
   For a single older message, prefer `git commit --fixup=reword:<sha>` (Git ≥2.32) + the autosquash command from step 2.
4. **Drop a commit:** `git rebase --onto <commit-before-bad> <bad-commit> <branch>`.
5. **Reorder / split** is genuinely interactive — if truly needed, do it as a series of `cherry-pick`s onto a fresh branch from `<base>` in the desired order.
6. **Replay over a moved base** (the common "rebase onto latest main"):
   ```bash
   git fetch origin && git rebase origin/main
   ```

## Verification

- `git log --oneline <base>..HEAD` shows the intended, clean history.
- `git diff backup/pre-rewrite HEAD` is **empty** — the rewrite changed history, not content. A non-empty diff means a fixup landed wrong; reset to the backup and retry.
- The branch still builds / tests pass — a rebase can produce a commit that compiles individually but breaks mid-history.

## Anti-cases

- Branch already pushed and shared → don't rewrite; add a new commit instead, or coordinate explicitly.
- `git diff` vs. the backup is non-empty → you changed content, not just history; abort and restart from the backup.
- Reaching for `rebase -i` out of habit → it hangs with no editor here; use the `--fixup` + `--autosquash` + `GIT_SEQUENCE_EDITOR` recipe.
