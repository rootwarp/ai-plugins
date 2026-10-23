---
description: 'Python code style: formatting, names, type hints, imports, code organization, and docstrings. Use when you write or review Python source code.'
name: python-code-style
---

# Code Style

Follow [PEP 8](https://peps.python.org/pep-0008/), [PEP 257](https://peps.python.org/pep-0257/),
and the [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html). If the
existing code is different, use the style of the existing code.

## Formatting and Imports

- `ruff format` controls the formatting, and the Ruff `I` rules control the import order. Do not
  format code manually.
- The line length is the `line-length` value in the project configuration. New projects from
  `/python:init-dir` use 79 (PEP 8). Do not reformat an existing project to a different limit.
- Use absolute imports (`from myapp.config import Settings`), not relative imports.
- When a module has more than about 500 lines, divide it. Put small related classes in one
  module.

## Type Hints

- Annotate each public function and method. Projects from `/python:init-dir` run mypy in strict
  mode.
- Target Python 3.11 and later. Use built-in generics (`list[str]`), `X | None` unions, and
  abstract types from `collections.abc`.
- In 3.12 and later, use the `type` statement for aliases and PEP 695 generics
  (`class Box[T]:`). In 3.11, use `TypeAlias` and `TypeVar`.
- Forward references: before 3.14, put the name in quotes or use
  `from __future__ import annotations`. In 3.14, annotations are evaluated only when they are
  used (PEP 649/749). Thus, you do not need either method.

## Data, Errors, and Functions

- Data containers: use a frozen `dataclass` for domain values. Use `pydantic.BaseModel` at I/O
  boundaries (see the `python-architecture` skill). Use composition, not inheritance.
- Raise specific exceptions. Give the domain its own exception hierarchy. Chain exceptions with
  `raise ... from e`. Catch only the exceptions that you can handle. Never use a bare `except:`.
- If a caller can easily put arguments in the wrong sequence, use keyword-only parameters
  (`def send(*, to: str, subject: str)`).
- Use guard clauses that return early. Do not use mutable default arguments.
- Use a context manager for each resource that needs cleanup.

## Docstrings and Comments

- Write docstrings for public modules, classes, and functions. Use the Google style (`Args:`,
  `Returns:`, `Raises:`), unless the project already uses a different style.
- In comments, tell why the code does something, not what it does.

## Comments and Messages

Write docstrings, error messages, and log messages in ASD-STE100 Simplified Technical English
(STE):

- Write short sentences. Use a maximum of 20 words in an instruction and 25 words in a
  description.
- Use the active voice and the simple present tense.
- Use one term for one concept. Do not use synonyms for the same concept.
- Do not write a noun cluster of more than three words.
- Do not use contractions, semicolons, or Latin abbreviations.

PEP 257 has priority over STE. The first line of a docstring is a one-line summary in the
imperative mood ("Return the user."), and it ends with a period.

For threads, asyncio, and processes, see the `python-concurrency` skill.

## Before Review

Run `uv run ruff format .`, `uv run ruff check .`, and `uv run mypy src/` (`.` for a flat
layout). Fix the problems that they report. See the `ruff` and `mypy` skills.
