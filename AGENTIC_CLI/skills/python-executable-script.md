---
name:        python-executable-script
description: Make a Python script runnable directly or as a standalone binary (shebang+chmod, PyInstaller, cx_Freeze).
version:     1
trigger_keywords: [python, executable, shebang, pyinstaller, cx_freeze, standalone, binary]
tools:       [bash, edit]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "make this script executable", "ship this as a standalone binary", "run it without typing `python`". First decide the target:

- **Run directly on a machine that has Python** → step 1 (shebang + chmod).
- **Ship to users without Python installed** → step 2 (PyInstaller) or step 3 (cx_Freeze).

Not a use case: packaging a library for `pip install` (that's `pyproject.toml`/`setup.py`, not a frozen binary); a project that already has an entry-point script.

## Prerequisites

- The script runs correctly with `python script.py` first — freezing a broken script just produces a broken binary.
- For frozen binaries: build **on the target OS**. PyInstaller/cx_Freeze do not cross-compile; build the Windows `.exe` on Windows, the Linux binary on Linux, etc.

## Procedure

### 1. Direct invocation on Unix-like systems

1. Add a shebang as the first line of the script:
   ```python
   #!/usr/bin/env python3
   ```
2. Mark it executable: `chmod +x script.py`
3. Run it: `./script.py`

### 2. Standalone binary with PyInstaller

1. `pip install pyinstaller`
2. From the script's directory: `pyinstaller --onefile script.py`
   - `--onefile` produces a single binary; without it, a folder with the binary plus dependencies.
3. The artifact lands in `dist/`.
4. If the script reads data files (images, templates), bundle them: `--add-data "src:dest"`.

### 3. Standalone binary with cx_Freeze

1. `pip install cx_Freeze`
2. Create `setup.py`:
   ```python
   from cx_Freeze import setup, Executable
   setup(
       name="MyApp", version="0.1",
       description="cx_Freeze build",
       executables=[Executable("script.py")],
   )
   ```
3. Build: `python setup.py build` → artifact under `build/`.

## Verification

- Step 1: `./script.py` runs without `python` prefixed and without a permission error.
- Steps 2–3: **run the binary on a machine that has no Python installed** (or a clean container) — this is the only real test that dependencies were bundled.
- Confirm bundled data files resolve at runtime, not just at build time.

## Anti-cases

- Need binaries for multiple OSes from one machine → not supported natively; use a VM, Docker, or a CI matrix per platform.
- The script is one file with only stdlib imports → a shebang (step 1) is enough; freezing is overkill.
