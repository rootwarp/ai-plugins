---
description: 'Run Clippy lints and fix the warnings. Use when the user asks you to lint Rust code.'
name: cargo-clippy
---

# /cargo-clippy

Run Clippy and fix the problems that it reports.

```bash
cargo clippy
```

The pre-commit gate uses the form that the `rust-code-style` skill requires. It covers all
targets and all features, and it makes warnings into errors:

```bash
cargo clippy --workspace --all-targets --all-features -- -D warnings
```

`cargo clippy --fix` applies the machine-applicable suggestions. It does not run on a dirty
tree. Commit or stash the changes first. If the user agrees, you can pass `--allow-dirty`.
