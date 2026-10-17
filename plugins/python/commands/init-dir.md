---
description: Initialize a Python project with uv, src layout, Ruff, mypy, and pytest
---

# /python:init-dir

Initialize a Python 3.11+ project with uv and modern tooling. The `pyproject.toml` tool
configuration in Step 5 is the canonical one for this plugin; the other skills refer to it.

If a command fails, stop and show the user the error rather than retrying with different flags,
deleting files, or installing tools yourself.

## Step 1: Preflight

```bash
uv --version                                      # uv must be installed
git rev-parse --is-inside-work-tree 2>/dev/null   # already inside a git repo?
```

- **uv missing:** stop and point the user to
  <https://docs.astral.sh/uv/getting-started/installation/>.
- **Target directory already contains a `pyproject.toml`:** stop and ask; `uv init` refuses to
  run in an existing project.
- **Inside a git repo:** pass `--vcs none` to `uv init` in Step 3 so no nested repo is created.

## Step 2: Gather Information

Ask for whatever `$ARGUMENTS` did not answer:

1. **Project type:** Application (recommended; `src/` layout with an entry point), Library
   (`src/` layout), or Simple Script (flat layout, no build system, no entry point)
2. **Project name** (package and directory name)
3. **License:** an SPDX expression such as `MIT` or `Apache-2.0`, or none
4. **Features:** web framework (FastAPI / Flask / none), CLI framework (Typer / Click / none;
   Application only), database (SQLAlchemy / asyncpg / none), type checker (mypy / pyright)

## Step 3: Create the Project with uv

uv creates the layout, so nothing has to be moved or deleted afterwards:

```bash
uv init --package --build-backend hatch <project-name>   # Application
uv init --lib --build-backend hatch <project-name>       # Library (adds py.typed)
uv init <project-name>                                   # Simple Script
cd <project-name>
```

## Step 4: Complete the Directory Structure

```bash
mkdir -p tests
touch tests/__init__.py tests/conftest.py
touch src/<package_name>/py.typed        # Application only; uv adds it for a Library
```

A Simple Script keeps uv's `main.py` as its entry point. Create `docs/` with the first document,
not before.

## Step 5: Configure pyproject.toml

Keep what uv generated in `[project]` (`name`, `version`, `readme`, `requires-python`, `authors`)
along with `[project.scripts]` and `[build-system]`; raise `requires-python` to `>=3.11` if uv
picked something older. Set only the license:

```toml
[project]
license = "MIT"          # the SPDX expression from Step 2; omit when "none"
```

Don't add `License ::` classifiers: PEP 639 supersedes them, and setuptools 77+ refuses to build
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

- **Simple Script:** omit `[tool.ruff.lint.isort]` and drop `--cov=<package_name>` from
  `addopts`.
- **pyright chosen:** replace `[tool.mypy]` with `[tool.pyright]` / `typeCheckingMode = "strict"`.
- **Async project** (FastAPI or asyncpg): add `asyncio_mode = "auto"` to
  `[tool.pytest.ini_options]` and `pytest-asyncio` to the dev tools in Step 11.
- **Pydantic in use** (FastAPI pulls it in): add `plugins = ["pydantic.mypy"]` to `[tool.mypy]`.

## Step 6: Add Selected Dependencies

```bash
uv add fastapi "uvicorn[standard]"   # FastAPI
uv add flask                         # Flask
uv add sqlalchemy alembic            # SQLAlchemy
uv add asyncpg                       # asyncpg
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

With Typer or Click, replace uv's single `[project.scripts]` entry rather than adding a second:

```toml
[project.scripts]
<project-name> = "<package_name>.cli:app"    # Typer
<project-name> = "<package_name>.cli:main"   # Click
```

## Step 7: Create Source Files

**`src/<package_name>/__init__.py`** (Application and Library):

```python
"""<project-name> - A brief description."""

__version__ = "0.1.0"
```

**`src/<package_name>/main.py`** (Application without a CLI framework). `uv init --package`
points `[project.scripts]` at `<package_name>:main`; since `main()` moves here, change that
entry to `<project-name> = "<package_name>.main:main"`.

```python
"""Main entry point."""


def main() -> None:
    """Run the application."""
    print("Hello, World!")


if __name__ == "__main__":
    main()
```

**Simple Script:** keep uv's `main.py`, annotated as `def main() -> None:` so strict mypy passes.

## Step 8: Create Test Files

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

## Step 9: Update .gitignore

`uv init` already wrote a `.gitignore` unless `--vcs none` was passed. Append only the entries it
lacks, or create the file:

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

## Step 10: Check .python-version

`uv init` writes `.python-version` (read by uv and pyenv). Keep it consistent with
`requires-python`; pin another interpreter with `uv python pin 3.12`.

## Step 11: Install Dev Tools

Swap `mypy` for `pyright` if it was chosen, and add `pytest-asyncio` for an async project:

```bash
uv add --dev pytest pytest-cov ruff mypy
uv sync
```

## Step 12: Verify and Show the Result

Show the structure with `tree` or `ls -la`, then run the initial checks (`mypy .` for a Simple
Script):

```bash
uv run ruff format .
uv run ruff check .
uv run mypy src/
uv run pytest
```
