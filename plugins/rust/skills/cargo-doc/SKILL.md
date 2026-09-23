---
description: 'Generate project docs with cargo doc and look up crate documentation on docs.rs. Use when verifying Rust APIs.'
name: cargo-doc
---

# /cargo-doc

Generate and view Rust documentation.

## Instructions

Generate documentation for the project's own crates (the default):

```bash
cargo doc --no-deps
```

Include dependencies only when you need their docs offline (much slower):

```bash
cargo doc
```

Output lands in `target/doc/<crate_name>/index.html`. Do not pass `--open` unless the user asks
to view the docs in a browser.

## Looking Up Crate Documentation

For crate documentation, use docs.rs. Take the lookup target from the request (`$ARGUMENTS` when
invoked as a command):

- Crate name to look up (e.g., `tokio`, `serde`, `axum`)
- Version (optional; e.g., `1.0.0`). Defaults to latest; prefer the version in `Cargo.lock` when
  the crate is already a dependency
- Item (optional): specific function, type, or module to focus on

### URL Structure

| Type | URL Format | Example |
|------|------------|---------|
| Latest | `https://docs.rs/{crate}` | `https://docs.rs/tokio` |
| Specific version | `https://docs.rs/{crate}/{version}` | `https://docs.rs/tokio/1.36.0` |
| Specific item | `https://docs.rs/{crate}/latest/{crate_ident}/{path}` | `https://docs.rs/serde_json/latest/serde_json/fn.from_str.html` |

`{crate_ident}` is the crate name with `-` replaced by `_` (the Rust identifier), e.g.
`https://docs.rs/tokio-util/latest/tokio_util/sync/index.html`.

### Execution

When invoked with a crate name, use WebFetch to retrieve the documentation:

```
WebFetch url="https://docs.rs/{crate}" prompt="Extract the documentation for this Rust crate. Include: 1) Crate overview 2) Key modules and their purposes 3) Important types and traits 4) Usage examples. If a specific item was requested, focus on that item's documentation."
```

## Standard Library Documentation

For standard library docs, use:

- Base URL: `https://doc.rust-lang.org/std/`
- Example: `https://doc.rust-lang.org/std/collections/struct.HashMap.html`

```
WebFetch url="https://doc.rust-lang.org/std/{path}" prompt="Extract the documentation for this Rust standard library item. Include its signature, description, methods, and examples."
```
