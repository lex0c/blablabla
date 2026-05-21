---
name:        bulk-edit-files
description: Apply a find-and-replace across many files safely — match preview, dry run, backup, apply, verify.
version:     1
trigger_keywords: [bulk edit, sed, find replace, mass rename, refactor across files, search and replace]
tools:       [bash]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "replace X with Y everywhere", "rename this string across the repo", "apply this change to every matching file". Use when the same textual change must land in **many files at once** and doing it one by one is impractical.

Prefer dedicated typed tools when available: an `Edit` with `replace_all` per file, or a tree-sitter rename for code symbols (`renomeia-simbolo`), is safer than raw `sed` because it is scoped and reviewable. Reach for this skill only for genuine cross-file textual sweeps.

Not a use case: renaming a code *symbol* (use a syntax-aware rename — `sed` matches inside strings/comments and breaks scope); a change that differs per file (not a uniform replace).

## Prerequisites

- A clean working tree, or a git repo — git *is* your backup and your verification (`git diff`).
- A precise match pattern. Vague patterns are the main failure mode.

## Procedure

1. **Preview the matches first — never skip this:**
   ```bash
   grep -rln 'PATTERN' .          # which files match
   grep -rc 'PATTERN' . | grep -v ':0'   # how many hits per file
   ```
   Read the file list. If it includes `.git/`, `node_modules/`, build dirs, or files you didn't expect — your pattern or scope is wrong. Narrow it (add `--include`, `--exclude-dir`, or a path) before going further.
2. **Dry-run the replacement** — show the diff without writing:
   ```bash
   grep -rl 'OLD' . | while read -r f; do
     diff <(cat "$f") <(sed 's/OLD/NEW/g' "$f")
   done
   ```
   Inspect every hunk. Confirm `OLD` is not matching as a substring of something larger, and not inside strings/comments where it shouldn't.
3. **Back up** if not in git: copy the target files, or `git stash`/commit so the pre-state is recoverable.
4. **Apply**, scoped to exactly the previewed file list:
   ```bash
   grep -rl 'OLD' . --include='*.py' | xargs sed -i 's/OLD/NEW/g'
   ```
   Use a regex delimiter other than `/` (e.g. `s|OLD|NEW|g`) if the text contains slashes. Escape regex metacharacters in `OLD`.
5. **Verify** (see below).

## Verification

- `grep -rc 'OLD' .` returns nothing (or only the intentional remaining hits) — the replacement count matches the preview from step 1.
- `git diff --stat` shows the **same file count** previewed in step 1 — no extra files touched, none missed.
- `git diff` hunks are all intended; nothing changed inside strings/comments/binaries by accident.
- The project still builds / tests pass — a textual sweep can produce syntactically valid but wrong code.

## Anti-cases

- Skipping the step 1 preview → `sed` silently edits `.git/`, lockfiles, or vendored deps; always scope first.
- `OLD` is a substring of a larger token → use word boundaries (`\bOLD\b`) or anchor the pattern.
- Renaming a code identifier → `sed` has no scope awareness; use a tree-sitter rename instead.
- `sed -i` with no git and no backup → an over-broad pattern is then unrecoverable.
