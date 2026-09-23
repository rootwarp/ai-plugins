---
description: 'Run Ruff lint and format via uv. Use when asked to lint or format Python code.'
name: ruff
---

# /python:ruff

Run Ruff linter and formatter on the Python project using uv.

## Instructions

Ruff reads `line-length` and rules from `pyproject.toml` (`[tool.ruff]`); the default line length
is 88. Projects created with `/python:init-dir` use 79.

### Linting

Check for lint issues (read-only):

```bash
uv run ruff check .
```

Auto-fix rewrites files. Run it only when the user asked for fixes, or ask first:

```bash
uv run ruff check --fix .
```

### Formatting

Check formatting without making changes:

```bash
uv run ruff format --check .
```

Format all Python files (rewrites files; only when the user asked to format):

```bash
uv run ruff format .
```

### Combined

When the user asked to fix and format:

```bash
uv run ruff check --fix . && uv run ruff format .
```

### If Ruff is not installed

`uv run ruff` fails with `Failed to spawn: ruff` when Ruff is not a dev dependency. Run it
one-off with `uvx ruff check .`, or ask the user before running `uv add --dev ruff`.

Analyze any lint warnings or errors and suggest fixes.
