---
description: 'Python project layout: src layout, package and module structure, entry points. Use when adding a module or package, or setting up a project.'
name: python-project-layout
---

# Project Layout

Follow [Python Packaging User Guide](https://packaging.python.org/) conventions and modern best
practices.

## Standard Project Layout

### src Layout (Recommended)

```
project/
├── pyproject.toml        # Project configuration
├── README.md
├── LICENSE
├── src/
│   └── mypackage/        # Package code
│       ├── __init__.py
│       ├── py.typed      # Marker for typed package
│       ├── main.py
│       └── utils.py
└── tests/
    ├── __init__.py
    ├── conftest.py       # pytest fixtures
    └── test_main.py
```

### Flat Layout (Simple scripts)

What `/python:init-dir` creates for a Simple Script: no package and no build system.

```
project/
├── pyproject.toml
├── README.md
├── main.py
└── tests/
    ├── __init__.py
    └── test_main.py
```

Once it needs more than a couple of modules or an installable entry point, move to the src
layout.

## Directory Purposes

| Directory | Purpose |
|-----------|---------|
| `src/<package>/` | Source code (src layout) |
| `tests/` | Test files |
| `docs/` | Documentation (create with the first document) |
| `scripts/` | Utility scripts |
| `data/` | Data files (not for secrets) |
| `.github/` | GitHub Actions workflows |

## pyproject.toml Configuration

Modern Python projects use `pyproject.toml` as the single configuration file. Project metadata:

```toml
[project]
name = "mypackage"
version = "0.1.0"
description = "A brief description"
readme = "README.md"
license = "MIT"
requires-python = ">=3.11"
authors = [
    { name = "Your Name", email = "you@example.com" }
]
keywords = ["keyword1", "keyword2"]
classifiers = [
    "Development Status :: 3 - Alpha",
    "Intended Audience :: Developers",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
]
dependencies = [
    "httpx>=0.27",
    "pydantic>=2.0",
]

[project.scripts]
myapp = "mypackage.main:main"

[project.urls]
Homepage = "https://github.com/username/mypackage"
Documentation = "https://mypackage.readthedocs.io"
Repository = "https://github.com/username/mypackage"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

Do not add `License ::` classifiers next to a `license` expression; PEP 639 supersedes them and
setuptools 77+ refuses to build.

For the tool configuration (`[tool.ruff]`, `[tool.mypy]`, `[tool.pytest.ini_options]`) and the
dev dependency group, use the canonical config in Step 5 of `/python:init-dir`; do not maintain a
diverging copy.

## Package Manager: uv

Use [uv](https://docs.astral.sh/uv/) for fast, reliable dependency management.

```bash
# Create new project
uv init myproject
cd myproject

# Add dependencies
uv add httpx pydantic

# Add dev dependencies
uv add --dev pytest ruff mypy

# Sync dependencies
uv sync

# Run commands
uv run python -m mypackage
uv run pytest

# Update dependencies
uv lock --upgrade
```

## Module Organization

```python
# src/mypackage/__init__.py
"""MyPackage - A brief description."""

from mypackage.core import process
from mypackage.models import Config, User

__version__ = "0.1.0"
__all__ = ["Config", "User", "process"]
```

```python
# src/mypackage/core.py
"""Core processing functions."""

from mypackage.models import Config

def process(config: Config) -> None:
    """Process data according to configuration."""
    ...
```

## Rules

- Use `src/` layout for libraries to prevent import confusion
- Flat layout is acceptable for simple scripts
- Always include `py.typed` marker for typed packages
- Keep `__init__.py` minimal; use for re-exports
- One logical concept per module
- Tests mirror source structure: `tests/test_<module>.py`

## Recommended Starting Structure

```
myproject/
├── pyproject.toml
├── README.md
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── py.typed        # Marker for typed package
│       └── main.py
└── tests/
    ├── __init__.py
    └── test_main.py
```

Add directories as needed, not preemptively.
