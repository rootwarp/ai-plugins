---
description: 'Run mypy type checking via uv and fix type errors. Use when asked to type-check Python code.'
name: mypy
---

# /python:mypy

Run type checking on the Python project using uv.

## Instructions

Pick the target from the layout: `src/` for a src-layout project, `.` for a flat or script
layout (no `src/` directory).

```bash
uv run mypy src/
uv run mypy .
```

Or check specific files:

```bash
uv run mypy src/<package_name>/main.py
```

For strict mode (already on when the project uses the `/python:init-dir` config):

```bash
uv run mypy --strict src/
```

### If mypy is not installed

`uv run mypy` fails with `Failed to spawn: mypy` when mypy is not a dev dependency. Do not add it
on your own: run it one-off with `uvx mypy src/` (third-party stubs will be missing), or ask the
user before running `uv add --dev mypy`.

### Missing stubs or untyped third-party packages

Do not pass `--ignore-missing-imports` globally; it hides real import errors. Install the stub
package if one exists (ask before `uv add --dev types-<package>`), otherwise scope the ignore:

```toml
[[tool.mypy.overrides]]
module = ["untyped_package.*"]
ignore_missing_imports = true
```

### Fixing errors

- Fix type errors with correct annotations or code changes
- Do not silence errors with `# type: ignore`, `cast`, or `Any` unless the error is a
  false positive; then use a scoped `# type: ignore[<code>]` with a comment explaining why
- If a fix would change runtime behavior, describe it and ask before applying
