---
name:        git-resolve-conflict
description: Resolve merge/rebase conflicts systematically — understand each side, pick deliberately, verify, continue.
version:     1
trigger_keywords: [git, merge conflict, rebase conflict, conflict markers, ours, theirs, rerere]
tools:       [bash, edit]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "resolve these conflicts", "the merge/rebase stopped on a conflict", "cherry-pick hit a conflict". Use whenever git halts with conflicting hunks during merge, rebase, cherry-pick, or stash pop.

Not a use case: deciding *whether* to merge or rebase (that's branch strategy); a clean merge with no conflict.

## Prerequisites

- An in-progress operation (`git status` shows `Unmerged paths` and the operation state).
- Know **which operation** you're in — it changes what "ours" and "theirs" mean (see step 2).

## Procedure

1. **Survey the damage:** `git status` lists unmerged paths; `git diff --name-only --diff-filter=U` is the bare list.
2. **Know the polarity** before touching anything:
   - **Merge**: `ours` = current branch, `theirs` = the branch being merged in.
   - **Rebase / cherry-pick**: **inverted** — `ours` = the branch you're rebasing *onto*, `theirs` = your commit being replayed. Getting this backwards silently discards work.
3. **Resolve each file.** Open it; conflict markers `<<<<<<<`, `=======`, `>>>>>>>` delimit the two sides. Decide deliberately:
   - Integrate both sides by hand (the usual correct answer), OR
   - Take one side wholesale: `git checkout --ours <file>` / `--theirs <file>` (mind step 2's polarity).
   - `git diff` (no args, during conflict) shows the combined diff to reason about.
4. **Mark resolved:** `git add <file>` for each. Removed-vs-modified conflicts: `git rm` or `git add` to declare intent.
5. **Continue the operation:**
   ```bash
   git merge --continue        # or: git rebase --continue / git cherry-pick --continue
   ```
   To bail out entirely: `git merge --abort` / `git rebase --abort` — returns to the pre-operation state cleanly.
6. **Optional, for repeated conflicts** (long rebase replaying similar hunks): enable `git config rerere.enabled true` once — git then records and replays your resolutions.

## Verification

- `git status` shows no unmerged paths; the operation completed (no lingering `MERGING`/`REBASE` state).
- **No conflict markers survived:** `git grep -nE '^(<<<<<<<|=======|>>>>>>>)' ` returns nothing.
- The result builds and tests pass — a syntactically merged file can still be semantically broken.
- `git diff <each-parent>...HEAD` looks intentional — no side's change silently vanished.

## Anti-cases

- Resolving a rebase conflict with merge-polarity assumptions → "ours/theirs" are inverted; you'll discard your own commit. Re-check step 2.
- `git add`-ing a file that still contains `=======` markers → the markers get committed as code; always run the grep in Verification.
- Blindly `--theirs` everything to make it stop → that's not resolution, it's deletion of one side's work.
