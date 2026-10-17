---
description: 'Generate project docs with cargo doc and look up crate documentation on docs.rs. Use when verifying Rust APIs.'
name: cargo-doc
---

# /cargo-doc

## Project Docs

Document the project's own crates (the default); output lands in
`target/doc/<crate_ident>/index.html`:

```bash
cargo doc --no-deps
```

Drop `--no-deps` only when dependency docs are needed offline (much slower). Pass `--open` only
when the user asks to view the docs in a browser.

## Looking Up a Crate or std

Take the crate, optional version, and optional item from the request (`$ARGUMENTS` when invoked
as a command) and WebFetch the page, asking for the overview, the requested item's signature and
description, and examples.

- Version: when the crate is already a dependency, use the version in `Cargo.lock`; otherwise
  `latest`.
- `{crate_ident}` is the crate name with `-` replaced by `_`.

| Target | URL |
|--------|-----|
| Crate | `https://docs.rs/{crate}/{version}` |
| Item | `https://docs.rs/{crate}/{version}/{crate_ident}/{path}` |
| std | `https://doc.rust-lang.org/std/{path}` |

Examples: `https://docs.rs/serde_json/latest/serde_json/fn.from_str.html`,
`https://docs.rs/tokio-util/latest/tokio_util/sync/index.html`,
`https://doc.rust-lang.org/std/collections/struct.HashMap.html`.
