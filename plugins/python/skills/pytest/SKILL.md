---
description: 'Run Python tests with pytest via uv and analyze failures. Use when asked to run or debug Python tests.'
name: pytest
---

# /python:pytest

Run tests for the Python project using uv.

## Instructions

Run tests with pytest:

```bash
uv run pytest
```

For verbose output:

```bash
uv run pytest -v
```

With coverage (requires `pytest-cov`; skip `--cov` if `addopts` already sets it):

```bash
uv run pytest --cov=<package_name> --cov-report=html
```

To run a specific test file or function:

```bash
uv run pytest tests/test_main.py
uv run pytest tests/test_main.py::test_function_name
```

To see print output:

```bash
uv run pytest -s
```

### If pytest is not installed

`uv run pytest` fails with `Failed to spawn: pytest` when pytest is not a dev dependency. Do not
add it on your own: ask the user before running `uv add --dev pytest` (and `pytest-cov` when
coverage is needed).

If tests fail, analyze the failures and suggest fixes.
