---
description: 'Python project layout: src layout, package and module structure, entry points. Use when adding a module or package, or setting up a project.'
name: python-project-layout
---

# Project Layout

Follow the [Python Packaging User Guide](https://packaging.python.org/). `pyproject.toml` is the
single configuration file; uv manages dependencies (see the `uv` skill).

## src Layout (default)

```text
project/
├── pyproject.toml
├── README.md
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── py.typed      # typed-package marker
│       └── main.py
└── tests/
    ├── __init__.py
    ├── conftest.py
    └── test_main.py
```

The src layout keeps tests running against the installed package rather than the working
directory. Grow it toward the layers in the `python-architecture` skill only when the code calls
for them.

## Flat Layout (simple scripts)

What `/python:init-dir` creates for a Simple Script: `main.py` at the root, a `tests/` directory,
no package and no build system. Move to the src layout once it needs more than a couple of
modules or an installable entry point.

## Rules

- Ship `py.typed` in every typed package.
- Keep `__init__.py` minimal: a docstring, `__version__`, and re-exports listed in `__all__`.
- One logical concept per module; avoid catch-all names such as `utils.py` or `helpers.py`.
- Tests mirror the source: `src/mypackage/billing.py` → `tests/test_billing.py`.
- Create `docs/`, `scripts/`, and `data/` with their first file, not before; `data/` never holds
  secrets.
- Declare console entry points in `[project.scripts]`.

## pyproject.toml

- `requires-python = ">=3.11"` or newer.
- `license` is an SPDX expression (`license = "MIT"`). Don't add `License ::` classifiers next to
  it: PEP 639 supersedes them, and setuptools 77+ refuses to build.
- Build backend: hatchling (what `/python:init-dir` uses).
- Tool configuration (`[tool.ruff]`, `[tool.mypy]`, `[tool.pytest.ini_options]`) comes from
  Step 5 of `/python:init-dir`, the canonical config, and the dev tools from its Step 11; don't
  keep a diverging copy.
