---
description: 'Run Python tests with pytest via uv and analyze failures. Use when asked to run or debug Python tests.'
name: pytest
---

# /python:pytest

Run the project's tests through uv:

```bash
uv run pytest                                   # whole suite
uv run pytest tests/test_main.py::test_name     # one file or one test
uv run pytest -v -s                             # verbose, show print output
uv run pytest --cov=<package_name> --cov-report=term-missing
```

`--cov` needs `pytest-cov`; leave it off when `addopts` already sets it (the `/python:init-dir`
config does).

`Failed to spawn: pytest` means pytest is not a dev dependency. Ask before running
`uv add --dev pytest` (plus `pytest-cov` when coverage is needed).

If tests fail, analyze the failures and suggest fixes.
