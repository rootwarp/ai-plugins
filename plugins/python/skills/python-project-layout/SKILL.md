---
description: 'Python project layout: src layout, package and module structure, and entry points. Use when you add a module or package, or set up a project.'
name: python-project-layout
---

# Project Layout

Follow the [Python Packaging User Guide](https://packaging.python.org/). `pyproject.toml` is the
only configuration file. uv manages the dependencies (see the `uv` skill).

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

With the src layout, the tests run against the installed package, not against the working
directory. Change it to the layers in the `python-architecture` skill only when the code needs
them.

## Flat Layout (simple scripts)

`/python:init-dir` creates this layout for a Simple Script. It has `main.py` at the root and a
`tests/` directory. It has no package and no build system. Change to the src layout when the
script needs more than two or three modules, or an installable entry point.

## Rules

- Include `py.typed` in each typed package.
- Keep `__init__.py` minimal: a docstring, `__version__`, and the re-exports in `__all__`.
- Put one logical concept in each module. Do not use general names such as `utils.py` or
  `helpers.py`.
- The test files have the same structure as the source files: `src/mypackage/billing.py` has
  the tests in `tests/test_billing.py`.
- Create `docs/`, `scripts/`, and `data/` when you add their first file, not before. Never put
  secrets in `data/`.
- Declare console entry points in `[project.scripts]`.

## pyproject.toml

- Set `requires-python = ">=3.11"` or a newer version.
- `license` is an SPDX expression (`license = "MIT"`). Do not add `License ::` classifiers with
  it. PEP 639 replaces these classifiers, and setuptools 77+ does not build the project if they
  are present.
- Build backend: use hatchling (`/python:init-dir` uses it).
- The tool configuration (`[tool.ruff]`, `[tool.mypy]`, `[tool.pytest.ini_options]`) comes from
  Step 5 of `/python:init-dir`, which is the canonical configuration. The dev tools come from
  Step 11. Do not keep a copy that is different.
