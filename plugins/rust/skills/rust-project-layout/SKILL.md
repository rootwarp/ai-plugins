---
description: 'Rust project layout: Cargo layout, module and crate organization, and workspace structure. Use when you add a module or crate, or when you set up a project.'
name: rust-project-layout
---

# Project Layout

Follow the [Cargo package layout](https://doc.rust-lang.org/cargo/guide/project-layout.html).
Add a directory when you add its first file. Do not add directories before you need them.

## Rules

- Commit `Cargo.lock` for binaries and also for libraries. This is the current Cargo guidance.
- Keep `src/main.rs` minimal. It parses the arguments and calls into `src/lib.rs`, which holds
  the logic.
- `lib.rs` holds the public API, the re-exports, and the crate docs. It defines types in
  submodules. The conventional modules are `error.rs` (error types and a `Result` alias),
  `config.rs`, and `cli.rs`.
- A module with submodules is `name.rs` + `name/`. Do not add `mod.rs` files. The one exception
  is `tests/common/mod.rs`. With this file, Cargo does not build the shared test helpers as a
  test.
- `tests/` covers only the public API. Examples must run with `cargo run --example <name>`.
- Put optional functionality behind feature flags. Enable only the dependency features that a
  crate uses. The `full` feature of `tokio` is for applications and quick prototypes, never for
  libraries.

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

- A virtual root has no package, so Cargo never compiles a `tests/` directory at the root. Put
  cross-crate tests in a member, for example `integration-tests`.
- A virtual root does not get the resolver from an edition. Set `resolver = "3"` explicitly.
  This is the edition-2024 default, and it is MSRV-aware.
- Start each member package name with the project name. A package that has only the name `core`
  has a conflict with the built-in crate.

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

Members inherit the values. Do not write the values again in each member:

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
