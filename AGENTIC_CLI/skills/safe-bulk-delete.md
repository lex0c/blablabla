---
name:        safe-bulk-delete
description: Delete many files matching a pattern without collateral damage — list, review, delete, verify.
version:     1
trigger_keywords: [bulk delete, rm, cleanup, remove files, find delete, purge]
tools:       [bash]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "delete all the X files", "clean up the build artifacts", "remove everything matching Z". Use when many files must be removed by a pattern and listing them by hand is impractical.

This is a **destructive, hard-to-reverse** operation — `rm` does not go to a trash can. The whole skill exists to put a review gate between the pattern and the deletion.

Not a use case: deleting a single known file (just delete it); removing files that are tracked in git and you want them gone from history (that's a history rewrite, not a delete).

## Prerequisites

- A precise match pattern, and a scoped root directory — never run from `/` or `$HOME` by accident.
- Know whether the targets are git-tracked: if so, `git rm` keeps the index consistent and the deletion is recoverable via git until committed.

## Procedure

1. **List the targets — never pipe `find` straight into `rm`:**
   ```bash
   find <root> -type f -name 'PATTERN' -print
   ```
   Read the entire list. Count it: append `| wc -l`. Confirm the count and the paths are what you expect — no surprise directories, no files outside `<root>`.
2. **Sanity-check the scope.** Does the list include anything tracked, anything under `.git/`, anything you did not intend? If yes, the pattern is wrong — refine `-name`, add `-path`/`-not -path`, narrow `<root>` — and return to step 1.
3. **Prefer recoverable deletion when possible:**
   - Git-tracked files: `git rm <files>` — recoverable until you commit.
   - Otherwise, move to a quarantine dir instead of deleting: `mkdir /tmp/quarantine && find ... -exec mv {} /tmp/quarantine/ \;` — inspect, then delete the quarantine later.
4. **Delete**, reusing the *exact* predicate from step 1:
   ```bash
   find <root> -type f -name 'PATTERN' -delete
   ```
   Use `find -delete` rather than `find -exec rm`/`xargs rm` — same predicate, no word-splitting on odd filenames. Never add `-r` to an `rm` unless directory removal is explicitly intended.
5. **Verify** (see below).

## Verification

- Re-run the step 1 `find`: it returns **nothing** — every target is gone.
- The files you intended to **keep** still exist (spot-check a few; `ls` the root).
- `git status` shows only the expected deletions — nothing tracked was removed unintentionally.
- The project still builds / runs — no live dependency was deleted.

## Anti-cases

- `find ... -delete` / `find ... | xargs rm` run before listing → no review gate; always do step 1 first.
- Pattern matches more than intended (e.g. `*.log` also hitting a config) → refine and re-list.
- Running from the wrong directory or with an unanchored pattern → scope `<root>` explicitly.
- Adding `rm -rf` "to be safe" → that is the opposite of safe; remove only files unless directories are the explicit target.
