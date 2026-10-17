---
description: 'Run Rust tests with cargo test and analyze failures. Use when asked to run or debug Rust tests.'
name: cargo-test
---

# /cargo-test

Run the tests, then diagnose any failures. The pre-commit gate in `rust-code-style` uses this
form:

```bash
cargo test --workspace --all-features
```

If `--all-features` fails because features are mutually exclusive, test the supported feature
sets explicitly (`--features a`, `--no-default-features --features b`).

- One test, or every test whose name contains the filter: `cargo test <name>`
- Show `println!`/`dbg!` output from passing tests: `cargo test -- --nocapture`
- Run tests marked `#[ignore]` (those needing external services): `cargo test -- --ignored`
