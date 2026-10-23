---
description: 'Run mypy type checks through uv and fix type errors. Use when the user asks you to type-check Python code.'
name: mypy
---

# /python:mypy

In a src-layout project, check `src/`. In a flat layout or a script layout (no `src/`
directory), check `.`. To check one module, give its file path:

```bash
uv run mypy src/
uv run mypy .
```

The `--strict` flag sets strict mode for one run. Projects that `/python:init-dir` creates
already set `strict = true`.

The message `Failed to spawn: mypy` tells you that mypy is not a dev dependency. Run it one time
with `uvx mypy src/`. In this mode, third-party stubs are not available. Alternatively, ask the
user before you run `uv add --dev mypy`.

## Missing stubs

Do not give `--ignore-missing-imports` for the full project, because it also hides real import
errors. If a stub package exists, install it. Ask the user before you run
`uv add --dev types-<package>`. If no stub package exists, set the ignore only for that package:

```toml
[[tool.mypy.overrides]]
module = ["untyped_package.*"]
ignore_missing_imports = true
```

## Type errors

- Fix the annotation or the code. Use `# type: ignore`, `cast`, or `Any` only for a false
  positive. In that case, use a scoped `# type: ignore[<code>]` and add a comment that gives the
  reason.
- If a fix changes the runtime behavior, describe the fix. Ask the user before you apply it.
