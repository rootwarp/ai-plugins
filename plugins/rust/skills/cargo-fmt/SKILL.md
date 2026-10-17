---
description: 'Format Rust sources with cargo fmt. Use when asked to format Rust code.'
name: cargo-fmt
---

# /cargo-fmt

Format every crate in the workspace:

```bash
cargo fmt --all
```

Check without writing (CI and the pre-commit gate):

```bash
cargo fmt --all --check
```
