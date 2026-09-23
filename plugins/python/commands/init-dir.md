---
description: Initialize a Python project with uv, src layout, Ruff, mypy, and pytest
---

# /python:init-dir

Initialize a standard Python project with uv and modern tooling. The `pyproject.toml` tool
configuration in Step 5 is the canonical one for this plugin; the other skills refer to it.

## Instructions

If any command below fails, stop and show the user the error. Do not retry with different flags,
delete files, or install tools on your own.

### Step 1: Preflight

```bash
uv --version                       # uv must be installed
git rev-parse --is-inside-work-tree 2>/dev/null   # already inside a git repo?
```

- **uv missing:** stop and point the user to
  <https://docs.astral.sh/uv/getting-started/installation/>. Do not install it yourself.
- **Target directory already exists** and contains a `pyproject.toml`: stop and ask the user;
  `uv init` refuses to run in an existing project.
- **Inside an existing git repo:** pass `--vcs none` to `uv init` in Step 3 so no nested repo is
  created.

### Step 2: Gather Information

Ask the user:
1. **Project type:**
   - Application (recommended) - executable with `src/` layout
   - Library - reusable package with `src/` layout
   - Simple Script - flat layout for small projects (no build system, no entry point)

2. **Project name** (used for package name and directory)

3. **License** (SPDX expression such as `MIT` or `Apache-2.0`, or none)

4. **Additional features:**
   - Web framework (FastAPI/Flask/none)
   - CLI framework (Typer/Click/none) - Application only
   - Database (SQLAlchemy/asyncpg/none)
   - Type checker (mypy/pyright)

### Step 3: Create Project with uv

Let uv create the layout, so nothing has to be moved or deleted afterwards:

```bash
# Application (src layout, with an entry point)
uv init --package --build-backend hatch <project-name>

# Library (src layout, with py.typed)
uv init --lib --build-backend hatch <project-name>

# Simple Script (flat layout, no build system)
uv init <project-name>

cd <project-name>
```

Add `--vcs none` when the preflight found an enclosing git repo.

### Step 4: Complete the Directory Structure

**Application:** uv created `src/<package_name>/__init__.py`.
```bash
mkdir -p tests
touch src/<package_name>/py.typed
touch tests/__init__.py tests/conftest.py
```

**Library:** uv created `src/<package_name>/__init__.py` and `py.typed`.
```bash
mkdir -p tests
touch tests/__init__.py tests/conftest.py
```

**Simple Script:** uv created `main.py`; keep it as the entry point.
```bash
mkdir -p tests
touch tests/__init__.py tests/conftest.py
```

Create `docs/` when the first document exists, not before.

### Step 5: Configure pyproject.toml

Keep what uv generated in `[project]` (`name`, `version`, `readme`, `requires-python`,
`authors` from git config) and its `[project.scripts]` and `[build-system]`. Set only the license:

```toml
[project]
license = "MIT"          # the SPDX expression from Step 2; omit when "none"
```

Do not add `License ::` classifiers; PEP 639 supersedes them, and setuptools 77+ refuses to build
when they appear next to a `license` expression.

Append the tool configuration:

```toml
[tool.ruff]
line-length = 79
# target-version is inferred from requires-python

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP", "B", "C4", "SIM", "RUF", "S"]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101"]    # pytest checks with assert

[tool.ruff.lint.isort]
known-first-party = ["<package_name>"]

[tool.mypy]
strict = true
# python_version defaults to the interpreter uv runs mypy with

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --strict-markers --cov=<package_name>"
markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests",
]
```

Variants:
- **Simple Script:** omit `[tool.ruff.lint.isort]`, drop `--cov=<package_name>` from `addopts`.
- **pyright chosen:** replace `[tool.mypy]` with
  `[tool.pyright]` / `typeCheckingMode = "strict"`.
- **Async project** (FastAPI or asyncpg): add `asyncio_mode = "auto"` to
  `[tool.pytest.ini_options]` and `pytest-asyncio` to the dev tools in Step 11.
- **Pydantic in use** (FastAPI pulls it in): add `plugins = ["pydantic.mypy"]` to `[tool.mypy]`.

### Step 6: Add Selected Dependencies

**FastAPI:**
```bash
uv add fastapi "uvicorn[standard]"
```

**Flask:**
```bash
uv add flask
```

**Typer** (Application): `uv add typer`, then create `src/<package_name>/cli.py`:
```python
"""Command-line interface."""

import typer

app = typer.Typer()


@app.command()
def main() -> None:
    """Run the application."""
    typer.echo("Hello, World!")
```

**Click** (Application): `uv add click`, then create `src/<package_name>/cli.py`:
```python
"""Command-line interface."""

import click


@click.command()
def main() -> None:
    """Run the application."""
    click.echo("Hello, World!")
```

With Typer or Click, replace uv's single `[project.scripts]` entry (do not add a second one):
```toml
[project.scripts]
<project-name> = "<package_name>.cli:app"    # Typer
<project-name> = "<package_name>.cli:main"   # Click
```

**SQLAlchemy:**
```bash
uv add sqlalchemy alembic
```

**asyncpg:**
```bash
uv add asyncpg
```

### Step 7: Create Source Files

**`src/<package_name>/__init__.py`:**
```python
"""<project-name> - A brief description."""

__version__ = "0.1.0"
```

**`src/<package_name>/main.py`** (Application without a CLI framework): `uv init --package`
points `[project.scripts]` at `<package_name>:main`; since `main()` moves here, change that entry
to `<project-name> = "<package_name>.main:main"`.
```python
"""Main entry point."""


def main() -> None:
    """Run the application."""
    print("Hello, World!")


if __name__ == "__main__":
    main()
```

**Simple Script:** keep uv's `main.py`, but annotate it as `def main() -> None:` so strict mypy
passes.

### Step 8: Create Test Files

**`tests/conftest.py`:**
```python
"""Pytest configuration and fixtures."""

import pytest


@pytest.fixture
def sample_data() -> dict[str, str]:
    """Provide sample test data."""
    return {"key": "value"}
```

**`tests/test_main.py`** (Application and Library):
```python
"""Tests for the package."""

from <package_name> import __version__


def test_version() -> None:
    """Test version is set."""
    assert __version__ == "0.1.0"
```

**`tests/test_main.py`** (Simple Script):
```python
"""Tests for main."""

import pytest

from main import main


def test_main_prints_greeting(capsys: pytest.CaptureFixture[str]) -> None:
    """Test main prints a greeting."""
    main()
    assert "Hello" in capsys.readouterr().out
```

### Step 9: Update .gitignore

`uv init` already wrote a `.gitignore` (unless `--vcs none`). Append only the entries it lacks;
create the file if it does not exist:

```gitignore
# Byte-compiled / optimized / DLL files
__pycache__/
*.py[cod]
*$py.class

# Distribution / packaging
dist/
build/
*.egg-info/

# Virtual environments
.venv/

# IDE / OS
.idea/
.vscode/
*.swp
.DS_Store

# Environment files
.env
.env.local

# Testing
.pytest_cache/
.coverage
htmlcov/
coverage.xml

# Tool caches
.mypy_cache/
.ruff_cache/
```

### Step 10: Check .python-version

`uv init` writes `.python-version` (read by uv and pyenv). Keep it consistent with
`requires-python`; pin another interpreter with `uv python pin 3.12`.

### Step 11: Install Dev Tools

Add the dev tools the user asked for as part of this setup (use `pyright` instead of `mypy` if
chosen; add `pytest-asyncio` for async projects), then install:

```bash
uv add --dev pytest pytest-cov ruff mypy
uv sync
```

### Step 12: Display Result

Show the created structure using `tree` or `ls -la` and confirm completion.

Run the initial checks (use `mypy .` for a Simple Script):
```bash
uv run ruff format .
uv run ruff check .
uv run mypy src/
uv run pytest
```

## Rules

- Use `src/` layout for packages to prevent import confusion
- Always include `py.typed` for typed packages
- Use `pyproject.toml` as single configuration file; Step 5 is the canonical tool config
- Python 3.11+ for modern features
- **Line length: 79 characters** for new projects (PEP 8); existing projects keep their setting
- Use `uv` for all dependency management
