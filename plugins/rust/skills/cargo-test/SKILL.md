---
description: 'Run Rust tests with cargo test and analyze failures. Use when asked to run or debug Rust tests.'
name: cargo-test
---

# /cargo-test

Run tests for the Rust project.

## Instructions

Run all tests (the pre-commit gate in `rust-code-style` uses this form):

```bash
cargo test --workspace --all-features
```

If `--all-features` fails because features are mutually exclusive, test the supported feature
sets explicitly (`--features a`, `--no-default-features --features b`).

To see `println!`/`dbg!` output from passing tests:

```bash
cargo test -- --nocapture
```

To run a specific test:

```bash
cargo test <test_name>
```

If tests fail, analyze the failures and suggest fixes.
