---
description: 'Run Clippy lints and fix warnings. Use when asked to lint Rust code.'
name: cargo-clippy
---

# /cargo-clippy

Run Clippy linter on the Rust project.

## Instructions

Run Clippy to check for common mistakes and improvements:

```bash
cargo clippy
```

For the pre-commit gate (all targets and features, warnings denied — the form the
`rust-code-style` skill requires):

```bash
cargo clippy --workspace --all-targets --all-features -- -D warnings
```

To fix automatically fixable issues (commit or stash first; `--fix` refuses a dirty tree unless
you add `--allow-dirty`):

```bash
cargo clippy --fix
```

Analyze any lint warnings or errors and suggest fixes.
