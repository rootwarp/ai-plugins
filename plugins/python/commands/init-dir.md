---
description: Initialize a Python project with uv, a src layout, Ruff, mypy, and pytest
---

# /python:init-dir

Initialize a Python 3.11+ project with uv and modern tools. The `pyproject.toml` tool
configuration in Step 5 is the canonical configuration for this plugin. The other skills refer
to it.

If a command fails, stop and show the error to the user. Do not try again with different flags.
Do not delete files, and do not install tools yourself.

## Step 1: Preflight

```bash
uv --version                                      # uv must be installed
git rev-parse --is-inside-work-tree 2>/dev/null   # already inside a git repo?
```

- **uv is not installed:** Stop. Refer the user to
  <https://docs.astral.sh/uv/getting-started/installation/>.
- **The target directory already contains a `pyproject.toml`:** Stop and ask the user. `uv init`
  does not run in an existing project.
- **The directory is in a git repo:** Give `--vcs none` to `uv init` in Step 3. This prevents a
  nested repo.

## Step 2: Get the Project Information

Ask the user for each item that `$ARGUMENTS` does not give:

1. **Project type:** Application (recommended, `src/` layout with an entry point), Library
   (`src/` layout), or Simple Script (flat layout, no build system, no entry point)
2. **Project name** (the name of the package and of the directory)
3. **License:** an SPDX expression such as `MIT` or `Apache-2.0`, or none
4. **Features:** web framework (FastAPI / Flask / none), CLI framework (Typer / Click / none,
   Application only), database (SQLAlchemy / asyncpg / none), type checker (mypy / pyright)

## Step 3: Create the Project with uv

uv creates the layout. Thus, you do not have to move or delete files after this step:

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

A Simple Script keeps the `main.py` from uv as its entry point. Create `docs/` when you add the
first document, not before.

## Step 5: Configure pyproject.toml

Keep the `[project]` fields that uv made (`name`, `version`, `readme`, `requires-python`,
`authors`). Also keep `[project.scripts]` and `[build-system]`. If uv selected a version older
than 3.11 for `requires-python`, change it to `>=3.11`. Set only the license:

```toml
[project]
license = "MIT"          # the SPDX expression from Step 2; omit when "none"
```

Do not add `License ::` classifiers. PEP 639 replaces them. Also, setuptools 77+ does not build
the project if they are present with a `license` expression.

Add the tool configuration at the end of the file:

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

- **Simple Script:** Do not add `[tool.ruff.lint.isort]`. Remove `--cov=<package_name>` from
  `addopts`.
- **The user selected pyright:** Replace `[tool.mypy]` with `[tool.pyright]` /
  `typeCheckingMode = "strict"`.
- **Async project** (FastAPI or asyncpg): Add `asyncio_mode = "auto"` to
  `[tool.pytest.ini_options]`. Add `pytest-asyncio` to the dev tools in Step 11.
- **The project uses Pydantic** (FastAPI includes it): Add `plugins = ["pydantic.mypy"]` to
  `[tool.mypy]`.

## Step 6: Add Selected Dependencies

```bash
uv add fastapi "uvicorn[standard]"   # FastAPI
uv add flask                         # Flask
uv add sqlalchemy alembic            # SQLAlchemy
uv add asyncpg                       # asyncpg
```

**Typer** (Application): Run `uv add typer`. Then create `src/<package_name>/cli.py`:

```python
"""Command-line interface."""

import typer

app = typer.Typer()


@app.command()
def main() -> None:
    """Run the application."""
    typer.echo("Hello, World!")
```

**Click** (Application): Run `uv add click`. Then create `src/<package_name>/cli.py`:

```python
"""Command-line interface."""

import click


@click.command()
def main() -> None:
    """Run the application."""
    click.echo("Hello, World!")
```

With Typer or Click, replace the one `[project.scripts]` entry that uv made. Do not add a second
entry:

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
sets `[project.scripts]` to `<package_name>:main`. Because `main()` moves to this file, change
that entry to `<project-name> = "<package_name>.main:main"`.

```python
"""Main entry point."""


def main() -> None:
    """Run the application."""
    print("Hello, World!")


if __name__ == "__main__":
    main()
```

**Simple Script:** Keep the `main.py` from uv. Annotate it as `def main() -> None:`, so that
strict mypy passes.

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

`uv init` writes a `.gitignore`, but not when you give `--vcs none`. Add only the entries that
are not in the file. If the file does not exist, create it:

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

`uv init` writes `.python-version`. uv and pyenv read this file. Make sure that it agrees with
`requires-python`. To pin a different interpreter, run `uv python pin 3.12`.

## Step 11: Install Dev Tools

If the user selected `pyright`, use it in place of `mypy`. For an async project, add
`pytest-asyncio`:

```bash
uv add --dev pytest pytest-cov ruff mypy
uv sync
```

## Step 12: Verify and Show the Result

Show the structure with `tree` or `ls -la`. Then run the initial checks. For a Simple Script,
run `mypy .`:

```bash
uv run ruff format .
uv run ruff check .
uv run mypy src/
uv run pytest
```
