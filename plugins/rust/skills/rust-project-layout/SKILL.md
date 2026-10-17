---
description: 'Rust project layout: Cargo layout, module and crate organization, workspace structure. Use when adding a module or crate, or setting up a project.'
name: rust-project-layout
---

# Project Layout

Follow the [Cargo package layout](https://doc.rust-lang.org/cargo/guide/project-layout.html).
Add directories when their first file arrives, not ahead of time.

## Rules

- Commit `Cargo.lock` for binaries and libraries alike (current Cargo guidance).
- Keep `src/main.rs` minimal: parse arguments and call into `src/lib.rs`, where the logic lives.
- `lib.rs` holds the public API, re-exports, and crate docs, and defines types in submodules.
  Conventional modules: `error.rs` (error types plus a `Result` alias), `config.rs`, `cli.rs`.
- A module with submodules is `name.rs` + `name/`. Don't add `mod.rs` files; the one exception is
  `tests/common/mod.rs`, which keeps shared test helpers from being built as a test.
- `tests/` covers the public API only. Examples must run with `cargo run --example <name>`.
- Optional functionality goes behind feature flags. Enable only the dependency features a crate
  uses (`tokio`'s `full` is for applications and quick prototypes, never libraries).

## Single Crate `Cargo.toml`

```toml
[package]
name = "my-project"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"              # MSRV
description = "A brief description"
license = "MIT OR Apache-2.0"
repository = "https://github.com/user/project"

[features]
default = []
# Each entry names a real feature or optional dependency:
# json = ["dep:serde_json"]

[profile.release]
lto = true
strip = true
```

## Workspaces

```text
project/
├── Cargo.toml              # virtual workspace manifest
└── crates/
    ├── core/               # package `project-core`
    ├── cli/                # package `project-cli`
    └── integration-tests/  # publish = false; cross-crate tests in its own tests/
```

- A virtual root has no package, so Cargo never compiles a root-level `tests/`. Put cross-crate
  tests in a member such as `integration-tests`.
- A virtual root does not inherit an edition's resolver, so set `resolver = "3"` explicitly (the
  edition-2024 default, MSRV-aware).
- Prefix member package names with the project name. A bare `core` clashes with the built-in
  crate.

```toml
# Cargo.toml (root)
[workspace]
resolver = "3"
members = ["crates/*"]

[workspace.package]
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
license = "MIT OR Apache-2.0"

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
```

Members inherit instead of repeating values:

```toml
# crates/core/Cargo.toml
[package]
name = "project-core"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true

[dependencies]
serde.workspace = true
```
