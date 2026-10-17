---
description: 'Run Clippy lints and fix warnings. Use when asked to lint Rust code.'
name: cargo-clippy
---

# /cargo-clippy

Run Clippy and fix what it reports.

```bash
cargo clippy
```

The pre-commit gate (the form the `rust-code-style` skill requires) covers every target and
feature and denies warnings:

```bash
cargo clippy --workspace --all-targets --all-features -- -D warnings
```

`cargo clippy --fix` applies the machine-applicable suggestions. It refuses a dirty tree, so
commit or stash first (or pass `--allow-dirty` when the user agrees).
