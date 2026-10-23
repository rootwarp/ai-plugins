---
description: 'Python test-driven development: Red-Green-Refactor, pytest layout, fixtures, and mocks. Use when you write tests or change the structure of a test module.'
name: python-tdd
---

# Test-Driven Development (TDD)

Use Red-Green-Refactor with pytest:

1. Write a test for the next behavior. Make sure that it fails.
2. Write the minimum code that makes the test pass.
3. Refactor while the tests stay green.

Before you commit, run the suite with `uv run pytest` (see the `pytest` skill).

## Layout and Names

- Put tests in `tests/`, with the same structure as the source (`tests/test_<module>.py`). Put
  shared fixtures in `tests/conftest.py`.
- Test one behavior in each test. Use the name `test_<unit>_<scenario>_<expected>`. If it is
  easier to read, put the tests of a unit in a `Test<Unit>` class.
- Use `@pytest.mark.parametrize` for tables of cases. Use `pytest.raises(..., match=...)` for
  errors. A fixture that needs teardown uses `yield`.

## Test Doubles

- Inject dependencies and give a fake. Alternatively, give a mock that you make with
  `create_autospec` (or `spec=`), so that it fails on calls that the real object rejects.
- Use `patch` only where injection is not possible. Patch the name where the code looks it up
  (`mypackage.email.smtp_connect`), not where the code defines it.
- Use `AsyncMock` for coroutines. Assert with `assert_awaited_once_with`.
- Tests never connect to real services. For httpx, mock the transport with `respx`.

## Async Tests

Async tests need `pytest-asyncio`. Ask the user before you run
`uv add --dev pytest-asyncio`. After you install it, set `asyncio_mode = "auto"` in
`[tool.pytest.ini_options]`. Then each `async def test_*` runs without a marker. If you set this
option before you install the package, pytest gives a warning about an unknown option. In the
default strict mode, mark each test with `@pytest.mark.asyncio`.

## Coverage

Get meaningful coverage. You do not have to get 100%. With the `dev-team` plugin, `code-writer`
requires a minimum of 70% coverage on changed code. This value is the same as its pre-review
gate:

```bash
uv run pytest --cov=mypackage --cov-report=term-missing --cov-fail-under=70
```

The canonical `[tool.pytest.ini_options]` (`testpaths`, `--strict-markers`, the `slow` and
`integration` markers, `--cov`) is in Step 5 of `/python:init-dir`.
