---
description: 'Format Rust source files with cargo fmt. Use when the user asks you to format Rust code.'
name: cargo-fmt
---

# /cargo-fmt

Format all crates in the workspace:

```bash
cargo fmt --all
```

To check the format without a change to the files, use this command. CI and the pre-commit gate
use it:

```bash
cargo fmt --all --check
```
