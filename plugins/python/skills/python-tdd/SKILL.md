---
description: 'Python test-driven development: Red-Green-Refactor, pytest layout, fixtures, mocking. Use when writing tests or restructuring a test module.'
name: python-tdd
---

# Test-Driven Development (TDD)

Red-Green-Refactor with pytest: write a failing test for the next behavior, make it pass with the
minimum code, then refactor with the tests green. Run the suite with `uv run pytest` (see the
`pytest` skill) before committing.

## Layout and Naming

- Tests live in `tests/`, mirroring the source (`tests/test_<module>.py`); shared fixtures go in
  `tests/conftest.py`.
- One behavior per test, named `test_<unit>_<scenario>_<expected>`; group a unit's tests in a
  `Test<Unit>` class when that reads better.
- Use `@pytest.mark.parametrize` for tables of cases and `pytest.raises(..., match=...)` for
  errors; fixtures that need teardown `yield`.

## Test Doubles

- Inject dependencies and pass a fake, or a mock built with `create_autospec` (or `spec=`) so it
  fails on calls the real object would reject.
- Use `patch` only where injection is impossible, and patch the name where it is looked up
  (`mypackage.email.smtp_connect`), not where it is defined.
- `AsyncMock` for coroutines; assert with `assert_awaited_once_with`.
- Tests never reach real services; for httpx, mock the transport with `respx`.

## Async Tests

Async tests need `pytest-asyncio`; ask before `uv add --dev pytest-asyncio`. Once it is
installed, set `asyncio_mode = "auto"` in `[tool.pytest.ini_options]` so every `async def test_*`
runs without a marker. Setting it earlier makes pytest warn about an unknown option. In the
default strict mode, mark each test with `@pytest.mark.asyncio`.

## Coverage

Aim for meaningful coverage, not 100%. With the `dev-team` plugin, `code-writer` enforces a
≥ 70% floor on changed code, matching its pre-review gate:

```bash
uv run pytest --cov=mypackage --cov-report=term-missing --cov-fail-under=70
```

The canonical `[tool.pytest.ini_options]` (`testpaths`, `--strict-markers`, `slow` and
`integration` markers, `--cov`) is in Step 5 of `/python:init-dir`.
