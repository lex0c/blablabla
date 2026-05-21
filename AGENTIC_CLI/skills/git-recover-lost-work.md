---
name:        git-recover-lost-work
description: Recover commits, branches, or stashes lost to a bad reset/rebase/checkout/branch-delete via reflog and fsck.
version:     1
trigger_keywords: [git, reflog, lost commit, hard reset, deleted branch, recover, dangling]
tools:       [bash]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "I lost my commits", "git reset --hard wiped my work", "I deleted the wrong branch", "the rebase ate a commit". Use when work that **was committed** (or stashed) is no longer reachable from any branch.

Not a use case: changes that were never committed and got wiped by `reset --hard` or `checkout .` — those are not in git's object DB and are unrecoverable here (try editor local history / IDE). A file deleted but the deletion committed → that's `git checkout <commit> -- <file>`, not this skill.

## Prerequisites

- Inside the git repo where the work was lost.
- The loss happened recently — reflog entries expire (90 days default for reachable, 30 for unreachable).
- Ideally, do not run `git gc` before recovering — it can prune dangling objects.

## Procedure

1. **Find the lost commit via reflog** — this is the move list of where `HEAD` has been:
   ```bash
   git reflog --date=iso          # HEAD history
   git reflog show <branchname>   # a specific branch's history
   ```
   Identify the SHA from just before the destructive command.
2. **If reflog has it**, recover by pointing a ref at the SHA:
   - Restore a branch: `git branch <recovered-name> <sha>`
   - Reset current branch back: `git reset --hard <sha>` (only if the current branch is the one to fix)
   - Cherry-pick just one lost commit: `git cherry-pick <sha>`
3. **If reflog is unhelpful** (e.g. commit was on a deleted branch with no reflog), scan dangling objects:
   ```bash
   git fsck --lost-found --no-reflogs
   ```
   Inspect candidates with `git show <sha>` / `git log <sha>`, then recover as in step 2.
4. **Lost stash**: `git fsck --no-reflogs | grep commit`, then `git stash apply <sha>` on the dangling commit, or `git log -g stash`.
5. **Confirm and re-anchor** the recovered work onto a branch so it stops being dangling.

## Verification

- `git log <recovered-ref>` shows the expected commits with the right messages and diffs.
- `git status` / `git show` confirms the file contents match what was lost.
- The recovered ref is a real branch (`git branch --list`), not just a detached SHA that the next `gc` could prune.

## Anti-cases

- Uncommitted, unstaged changes wiped by `reset --hard` → not in the object DB; this skill cannot help.
- "Lost" commit is actually still on another branch → use `git branch --contains <sha>` first; it may not be lost at all.
- Don't `git gc --prune=now` while diagnosing — it destroys exactly what you're trying to recover.
