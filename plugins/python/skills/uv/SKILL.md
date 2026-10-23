---
description: 'Manage Python dependencies with uv (add, remove, sync, update). Use when you change Python dependencies.'
name: uv
---

# /python:uv

Use uv to change dependencies. Do not edit `pyproject.toml` manually. This keeps `uv.lock` in
sync with `pyproject.toml`.

| Task | Command |
|------|---------|
| Install all packages from `uv.lock` | `uv sync` |
| Add runtime dependencies | `uv add httpx "pydantic>=2"` |
| Add a dev dependency | `uv add --dev <package>` |
| Remove a dependency | `uv remove <package>` (`--dev` for a dev dependency) |
| Upgrade one package | `uv lock --upgrade-package <package> && uv sync` |
| Upgrade all packages | `uv lock --upgrade && uv sync` |
| Run in the project environment | `uv run python -m <module>` |

Ask the user before you upgrade all packages, because this can bring in many unrelated changes.
