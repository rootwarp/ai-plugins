---
description: 'Do a fast type check with cargo check. It does not make binaries. Use when you must find Rust errors quickly.'
name: cargo-check
---

# /cargo-check

Do a type check without code generation. Then fix the errors:

```bash
cargo check
```

Plain `cargo check` skips tests, examples, and benches. To include them, use this command:

```bash
cargo check --all-targets
```
