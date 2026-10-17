---
description: 'Manage Python dependencies with uv (add, remove, sync, update). Use when changing Python dependencies.'
name: uv
---

# /python:uv

Change dependencies through uv instead of editing `pyproject.toml` by hand, so `uv.lock` stays
in sync.

| Task | Command |
|------|---------|
| Install everything from `uv.lock` | `uv sync` |
| Add runtime dependencies | `uv add httpx "pydantic>=2"` |
| Add a dev dependency | `uv add --dev <package>` |
| Remove a dependency | `uv remove <package>` (`--dev` for a dev dependency) |
| Upgrade one package | `uv lock --upgrade-package <package> && uv sync` |
| Upgrade everything | `uv lock --upgrade && uv sync` |
| Run inside the project environment | `uv run python -m <module>` |

Ask before upgrading everything, because it can pull in many unrelated changes.
