---
description: 'Run mypy type checking via uv and fix type errors. Use when asked to type-check Python code.'
name: mypy
---

# /python:mypy

Check `src/` in a src-layout project and `.` in a flat or script layout (no `src/` directory);
pass a file path to check one module:

```bash
uv run mypy src/
uv run mypy .
```

`--strict` enables strict mode for one run; projects created with `/python:init-dir` already set
`strict = true`.

`Failed to spawn: mypy` means mypy is not a dev dependency. Run it one-off with `uvx mypy src/`
(third-party stubs will be missing), or ask before running `uv add --dev mypy`.

## Missing stubs

Don't pass `--ignore-missing-imports` globally, since it also hides real import errors. Install
the stub package when one exists (ask before `uv add --dev types-<package>`); otherwise scope the
ignore:

```toml
[[tool.mypy.overrides]]
module = ["untyped_package.*"]
ignore_missing_imports = true
```

## Fixing errors

- Fix the annotation or the code. Reach for `# type: ignore`, `cast`, or `Any` only for a
  false positive, and then as a scoped `# type: ignore[<code>]` with a comment saying why.
- If a fix would change runtime behavior, describe it and ask before applying it.
