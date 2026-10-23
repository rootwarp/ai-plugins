---
description: 'Run the Ruff linter and formatter through uv. Use when the user asks you to lint or format Python code.'
name: ruff
---

# /python:ruff

Ruff reads its rules and `line-length` from `[tool.ruff]` in `pyproject.toml`. The default
line length is 88. Projects that `/python:init-dir` creates use 79.

```bash
uv run ruff check .              # lint, read-only
uv run ruff format --check .     # formatting, read-only
uv run ruff check --fix .        # rewrites files
uv run ruff format .             # rewrites files
```

Run the commands that rewrite files only if the user asked for fixes or for formatting. If not,
report the findings and ask the user. To fix and format in one command, run
`uv run ruff check --fix . && uv run ruff format .`.

The message `Failed to spawn: ruff` tells you that Ruff is not a dev dependency. Run it one time
with `uvx ruff check .`. Alternatively, ask the user before you run `uv add --dev ruff`.

Analyze the warnings and errors that remain, and recommend fixes.
