---
description: 'Generate project docs with cargo doc and look up crate documentation on docs.rs. Use when you must verify Rust APIs.'
name: cargo-doc
---

# /cargo-doc

## Project Docs

By default, generate docs for the crates of the project only. The output goes to
`target/doc/<crate_ident>/index.html`:

```bash
cargo doc --no-deps
```

Remove `--no-deps` only if you need the docs of dependencies offline. This is much slower.
Pass `--open` only if the user asks to see the docs in a browser.

## Find Docs for a Crate or std

Get the crate, the optional version, and the optional item from the request. When the user
starts this skill as a command, they are in `$ARGUMENTS`. Use WebFetch to get the page. Ask for
the overview, the signature and description of the item, and examples.

- Version: If the crate is already a dependency, use the version in `Cargo.lock`. If not, use
  `latest`.
- `{crate_ident}` is the crate name with each `-` changed to `_`.

| Target | URL |
|--------|-----|
| Crate | `https://docs.rs/{crate}/{version}` |
| Item | `https://docs.rs/{crate}/{version}/{crate_ident}/{path}` |
| std | `https://doc.rust-lang.org/std/{path}` |

Examples: `https://docs.rs/serde_json/latest/serde_json/fn.from_str.html`,
`https://docs.rs/tokio-util/latest/tokio_util/sync/index.html`,
`https://doc.rust-lang.org/std/collections/struct.HashMap.html`.
