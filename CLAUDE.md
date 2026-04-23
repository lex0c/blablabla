# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository type

This is a **personal knowledge base**, not a software project. It is a flat collection of Markdown notes at the repository root — one file per topic, covering software engineering, security, AI, hardware, mechanics, games, and other subjects the owner studies or references. There is no build system, no dependencies, no tests, no CI. Every change is an edit to a `.md` file.

Because there is no code to run or compile, there are no build/lint/test commands. Do not invent any.

## File conventions

- Filenames are `SCREAMING_SNAKE_CASE.md` at the repo root (e.g., `REVERSE_ENGINEERING.md`, `AI_SYSTEMS_DESIGN.md`). Preserve this when creating new notes.
- Content language is a mix of **Portuguese and English** — match the language already used inside a given file rather than translating it. Many files mix both intentionally (Portuguese prose around English technical terms).
- Notes are self-contained; there is no cross-file linking convention or generated index. Don't invent one.
- Heading style is standard Markdown (`#`, `##`, `###`). Code fences use language tags where relevant.

## Commit conventions

Commits in this repo follow a strict verb-prefix pattern visible in `git log`:

- `Create FOO.md` — new file
- `Create FOO.md, BAR.md, BAZ.md` — multiple new files in one commit
- `Update FOO.md` / `Update FOO.md, BAR.md` — edits
- `Delete FOO.md` / `Delete FOO.md, BAR.md` — removals

Rules:
- **Title Case verb** (`Create`, `Update`, `Delete`) — never ALL CAPS, never lowercase.
- List the affected filenames, comma-separated.
- No scope, no body, no conventional-commits prefix (`feat:`/`fix:`), no trailer — just the one-line subject.
