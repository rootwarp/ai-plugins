---
description: 'Fast type-check with cargo check without producing binaries. Use for quick Rust error checking.'
name: cargo-check
---

# /cargo-check

Type-check without code generation, then fix any errors:

```bash
cargo check
```

Plain `cargo check` skips tests, examples, and benches; include them with:

```bash
cargo check --all-targets
```
