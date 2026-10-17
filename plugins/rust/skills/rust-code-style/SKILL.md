---
description: 'Rust code style: formatting, naming, code organization, trait and derive conventions. Use when writing or reviewing Rust source.'
name: rust-code-style
---

# Code Style

Follow [The Rust Style Guide](https://doc.rust-lang.org/nightly/style-guide/), the [Rust API
Guidelines](https://rust-lang.github.io/api-guidelines/), and Clippy. The rules below are the
project's choices on top of those.

## Naming and Organization

- Package names are `kebab-case` in `Cargo.toml` (`my-crate`, imported as `my_crate`); prefer a
  single word.
- Group imports std, external crates, then `crate::`, separated by blank lines (stable rustfmt
  does not regroup them for you).
- Default to private, then `pub(crate)`; make an item `pub` only when it is part of the API.
  Re-export the public API from `lib.rs` and define the types in submodules.
- Keep traits small and compose them with supertrait bounds (see the `rust-architecture`
  skill). Derive `Debug` on every public type, plus `Clone`, `PartialEq`, `Eq`, `Hash` where
  they make sense.
- Take borrowed parameters (`&str`, `&[T]`, `impl AsRef<Path>`) and return owned values; use
  `Cow<'_, str>` when ownership is conditional. Clone explicitly, where the cost is visible.

## Errors

- Libraries define typed errors with `thiserror`; binaries use `anyhow` and attach context with
  `.context(...)`. Put the failing input (path, id) in the error, not only the cause.
- No `.unwrap()` or `.expect()` on fallible operations in library code; return a `Result`. An
  `.expect("why this cannot fail")` on a true invariant is fine. Lock results are the usual
  exception (see `rust-concurrency`).
- Panic only for bugs, never for expected failures such as bad input or I/O errors.

## Documentation

- Document every public item with `///`, and each module or crate with `//!`. Add `# Errors`,
  `# Panics`, and `# Safety` sections whenever they apply.
- Comments explain why; the code says what.
- Doc examples are compiled and run by `cargo test`, so they are part of the quality gate. Hide
  setup lines with `# `, end an example that uses `?` with `# Ok::<(), E>(())`, and mark
  examples that touch the filesystem or network `no_run` (still compiled, not executed). Use
  `ignore` only for code that cannot compile in isolation.

Concurrency, testing, and `unsafe` live in the `rust-concurrency`, `rust-tdd`, and
`rust-security` skills.

## Quality Gate

Run this before every commit or PR, and fix whatever fails before committing:

```bash
cargo fmt --all --check \
  && cargo clippy --workspace --all-targets --all-features -- -D warnings \
  && cargo test --workspace --all-features
```

- Fix lints rather than silencing them. When one must stay, prefer
  `#[expect(clippy::lint_name, reason = "...")]` (Rust 1.81+) over `#[allow]`: it warns once
  the lint no longer fires.
- `--workspace` matters when the root manifest is also a package (otherwise only the root crate
  is checked); it is harmless in a single crate.
- Mutually exclusive features make `--all-features` fail to build. Replace it with the
  supported feature sets, or `cargo hack --each-feature` if `cargo-hack` is installed (ask
  before installing it).
- If the project has a `deny.toml`, also run `cargo deny check` (see `rust-security`).
