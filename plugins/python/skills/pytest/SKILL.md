---
description: 'Run Python tests with pytest through uv and analyze the failures. Use when the user asks you to run or debug Python tests.'
name: pytest
---

# /python:pytest

Run the tests of the project through uv:

```bash
uv run pytest                                   # whole suite
uv run pytest tests/test_main.py::test_name     # one file or one test
uv run pytest -v -s                             # verbose, show print output
uv run pytest --cov=<package_name> --cov-report=term-missing
```

The `--cov` flag needs `pytest-cov`. Do not give this flag if `addopts` already sets it. The
`/python:init-dir` configuration sets it.

The message `Failed to spawn: pytest` tells you that pytest is not a dev dependency. Ask the
user before you run `uv add --dev pytest`. If the user needs coverage, also add `pytest-cov`.

If tests fail, analyze the failures and recommend fixes.
