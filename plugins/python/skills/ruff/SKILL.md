---
description: 'Run Ruff lint and format via uv. Use when asked to lint or format Python code.'
name: ruff
---

# /python:ruff

Ruff reads its rules and `line-length` from `[tool.ruff]` in `pyproject.toml` (default 88;
projects created with `/python:init-dir` use 79).

```bash
uv run ruff check .              # lint, read-only
uv run ruff format --check .     # formatting, read-only
uv run ruff check --fix .        # rewrites files
uv run ruff format .             # rewrites files
```

Run the rewriting commands only when the user asked for fixes or formatting; otherwise report
the findings and ask. To fix and format together:
`uv run ruff check --fix . && uv run ruff format .`

`Failed to spawn: ruff` means Ruff is not a dev dependency. Run it one-off with
`uvx ruff check .`, or ask before running `uv add --dev ruff`.

Analyze any remaining warnings or errors and suggest fixes.
