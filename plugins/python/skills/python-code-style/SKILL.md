---
description: 'Python code style: formatting, naming, typing, imports, code organization. Use when writing or reviewing Python source.'
name: python-code-style
---

# Code Style

Follow [PEP 8](https://peps.python.org/pep-0008/), [PEP 257](https://peps.python.org/pep-0257/),
and the [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html). Where the
existing code differs, match the existing code.

## Formatting and Imports

- `ruff format` owns formatting and Ruff's `I` rules own import order; don't hand-format.
- Line length is the project's configured `line-length`. New projects from `/python:init-dir`
  use 79 (PEP 8); don't reformat an existing project to a different limit.
- Prefer absolute imports (`from myapp.config import Settings`) over relative ones.
- Split a module once it passes about 500 lines; group small related classes in one module.

## Type Hints

- Annotate every public function and method; projects from `/python:init-dir` run strict mypy.
- Target 3.11+: built-in generics (`list[str]`), `X | None` unions, abstract types from
  `collections.abc`.
- 3.12+: the `type` statement for aliases and PEP 695 generics (`class Box[T]:`); on 3.11 use
  `TypeAlias` and `TypeVar`.
- Forward references: below 3.14, quote the name or use `from __future__ import annotations`;
  3.14 defers annotation evaluation (PEP 649/749), so neither is needed.

## Data, Errors, and Functions

- Data containers: frozen `dataclass` for domain values, `pydantic.BaseModel` at I/O boundaries
  (see the `python-architecture` skill). Prefer composition over inheritance.
- Raise specific exceptions; give the domain its own exception hierarchy; chain with
  `raise ... from e`; catch only what you can handle, never a bare `except:`.
- Use keyword-only parameters (`def send(*, to: str, subject: str)`) when arguments are easy to
  swap; return early with guard clauses; no mutable default arguments.
- Use context managers for every resource that needs cleanup.

## Docstrings and Comments

- Docstrings on public modules, classes, and functions, Google style (`Args:`, `Returns:`,
  `Raises:`) unless the project already uses another style.
- Comments explain why, not what.

For threads, asyncio, and processes, see the `python-concurrency` skill.

## Before Review

Run `uv run ruff format .`, `uv run ruff check .`, and `uv run mypy src/` (`.` for a flat
layout), and fix what they report; see the `ruff` and `mypy` skills.
