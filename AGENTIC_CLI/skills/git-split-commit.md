---
name:        git-split-commit
description: Break a messy working tree or an oversized commit into focused, atomic commits via add -p and reset.
version:     1
trigger_keywords: [git, add -p, atomic commit, split commit, stage hunks, patch mode]
tools:       [bash]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "split this into separate commits", "this commit does too much", "make this PR reviewable". Use when changes that belong in **distinct logical units** are mixed in one working tree or one commit.

Not a use case: a commit that is already atomic but large (size alone is not the problem — coherence is); reordering existing well-formed commits (use `git-rewrite-history`).

## Prerequisites

- For an unstaged working tree: nothing extra.
- For splitting an *existing* commit: it must be local/unpushed, and the tree clean otherwise.
- A safety net: `git branch backup/pre-split`.

## Procedure

### A. Working tree not yet committed

1. **Reset staging** so nothing is staged: `git restore --staged .` (or start fresh).
2. **Stage one logical unit at a time** with patch mode:
   ```bash
   git add -p <files>      # y/n per hunk; s to split a hunk; e to edit it
   ```
   If a single file mixes concerns, `s` splits the hunk; `e` lets you hand-pick lines.
3. **Commit that unit**, then repeat step 2 for the next: `git commit -m "..."`.
4. Whole-file units that belong together: `git add <file>` directly.

### B. Splitting an existing commit

1. `git branch backup/pre-split`.
2. Move the commit's changes back into the working tree, keeping history before it:
   ```bash
   git reset HEAD~1        # mixed reset: commit undone, changes unstaged
   ```
3. Now apply section A (steps 2–3) to re-commit in focused pieces.
4. If the commit to split is **not** the latest, first `git rebase` so it becomes `HEAD` (or use `git-rewrite-history`), then split, then the later commits replay.

## Verification

- `git diff backup/pre-split HEAD` is **empty** — the split changed commit boundaries, not content.
- `git status` is clean — no hunk was left behind unstaged/uncommitted.
- Each commit builds and ideally passes tests on its own (`git stash` the rest and check, or `git rebase --exec`).
- `git log --oneline` shows each commit with a message describing one concern.

## Anti-cases

- `git diff` vs. the backup is non-empty → a hunk was dropped or duplicated; reset to the backup and redo.
- Splitting just to hit a commit-count target → atomicity is about logical coherence, not arithmetic.
- The commit is already pushed and shared → splitting rewrites history; coordinate or leave it.
