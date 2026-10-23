---
description: 'Run Rust tests with cargo test and analyze the failures. Use when the user asks you to run or debug Rust tests.'
name: cargo-test
---

# /cargo-test

Run the tests. Then find the cause of each failure. The pre-commit gate in `rust-code-style` uses
this form:

```bash
cargo test --workspace --all-features
```

Some features cannot be used together. If `--all-features` fails for this reason, test each
supported feature set explicitly (`--features a`, `--no-default-features --features b`).

- To run one test, or all tests with names that contain the filter: `cargo test <name>`
- To show the `println!` and `dbg!` output of tests that pass: `cargo test -- --nocapture`
- To run the tests marked `#[ignore]`, which need external services: `cargo test -- --ignored`
