---
description: 'Rust code style: formatting, names, code organization, trait and derive conventions, and comments. Use when you write or review Rust source code.'
name: rust-code-style
---

# Code Style

Follow [The Rust Style Guide](https://doc.rust-lang.org/nightly/style-guide/), the [Rust API
Guidelines](https://rust-lang.github.io/api-guidelines/), and Clippy. The rules below are the
decisions that this project adds to them.

## Names and Organization

- Write package names in `kebab-case` in `Cargo.toml` (`my-crate`, imported as `my_crate`). Use
  a single word if possible.
- Put imports in three groups in this sequence: std, external crates, and `crate::`. Put a blank
  line between the groups. Stable rustfmt does not make these groups for you.
- Make items private by default, then `pub(crate)`. Make an item `pub` only when it is part of
  the API. Re-export the public API from `lib.rs`, and define the types in submodules.
- Keep traits small, and combine them with supertrait bounds (see the `rust-architecture`
  skill). Derive `Debug` on each public type. Also derive `Clone`, `PartialEq`, `Eq`, and `Hash`
  where they are useful.
- Take borrowed parameters (`&str`, `&[T]`, `impl AsRef<Path>`) and return owned values. Use
  `Cow<'_, str>` when the ownership depends on a condition. Clone explicitly, where the reader
  can see the cost.

## Errors

- In libraries, define typed errors with `thiserror`. In binaries, use `anyhow` and add context
  with `.context(...)`. Put the failed input (path, id) in the error, not only the cause.
- In library code, do not use `.unwrap()` or `.expect()` on operations that can fail. Return a
  `Result`. You can use `.expect("why this cannot fail")` on a true invariant. Lock results are
  the usual exception (see `rust-concurrency`).
- Use `panic` only for bugs. Do not use it for expected failures, for example bad input or I/O
  errors.

## Documentation

- Document each public item with `///`, and each module or crate with `//!`. Add `# Errors`,
  `# Panics`, and `# Safety` sections when they apply.
- Comments tell why. The code tells what.
- `cargo test` compiles and runs doc examples, so they are part of the quality gate. Hide setup
  lines with `# `. End an example that uses `?` with `# Ok::<(), E>(())`. Mark examples that use
  the filesystem or the network `no_run`, so they compile but do not run. Use `ignore` only for
  code that cannot compile alone.

## Comments and Messages

Write doc comments, error messages, and log messages in ASD-STE100 Simplified Technical English
(STE):

- Write short sentences. Use a maximum of 20 words in an instruction and 25 words in a
  description.
- Use the active voice and the simple present tense.
- Use one term for one concept. Do not use synonyms for the same concept.
- Do not write a noun cluster of more than three words.
- Do not use contractions, semicolons, or Latin abbreviations.

The rustdoc conventions have priority over STE. The first line of a doc comment is a one-line
summary. An error message (the `Display` output of an error type) starts with a lowercase letter
and has no final period (Rust API guidelines C-GOOD-ERR).

The `rust-concurrency`, `rust-tdd`, and `rust-security` skills give the rules for concurrency,
tests, and `unsafe`.

## Quality Gate

Run this command before each commit or PR. If a step fails, fix the problem before you commit:

```bash
cargo fmt --all --check \
  && cargo clippy --workspace --all-targets --all-features -- -D warnings \
  && cargo test --workspace --all-features
```

- Fix lints. Do not silence them. If a lint must stay, use
  `#[expect(clippy::lint_name, reason = "...")]` (Rust 1.81+), not `#[allow]`. It gives a
  warning when the lint no longer fires.
- `--workspace` is important when the root manifest is also a package. Without it, Cargo checks
  only the root crate. In a single crate, it does no harm.
- If features cannot be used together, `--all-features` fails to build. Replace it with the
  supported feature sets. If `cargo-hack` is installed, you can use `cargo hack --each-feature`.
  Ask before you install it.
- If the project has a `deny.toml`, also run `cargo deny check` (see `rust-security`).
