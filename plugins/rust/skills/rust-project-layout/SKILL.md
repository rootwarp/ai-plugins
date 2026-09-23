---
description: 'Rust project layout: Cargo layout, module and crate organization, workspace structure. Use when adding a module or crate, or setting up a project.'
name: rust-project-layout
---

# Project Layout

Follow [Cargo project conventions](https://doc.rust-lang.org/cargo/guide/project-layout.html) and
community best practices.

## Standard Cargo Layout

| Path | Purpose |
|------|---------|
| `Cargo.toml` | Package manifest |
| `Cargo.lock` | Dependency lock file (commit it — current guidance for binaries and libraries) |
| `src/lib.rs` | Library crate root |
| `src/main.rs` | Binary crate root (single binary) |
| `src/bin/` | Additional binary crates |
| `tests/` | Integration tests |
| `benches/` | Benchmarks |
| `examples/` | Example code |
| `build.rs` | Build script |

## Source Organization

| Path | Purpose |
|------|---------|
| `src/lib.rs` | Public API, re-exports, crate-level docs |
| `src/error.rs` | Error types and `Result` alias |
| `src/config.rs` | Configuration types |
| `src/*/mod.rs` | Module definitions |

## Workspace Structure

For multi-crate projects:

```
project/
├── Cargo.toml          # Workspace manifest
├── crates/
│   ├── core/           # Core library
│   │   ├── Cargo.toml
│   │   └── src/
│   ├── cli/            # CLI binary
│   │   ├── Cargo.toml
│   │   └── src/
│   └── server/         # Server binary
│       ├── Cargo.toml
│       └── src/
├── tests/              # Workspace-level integration tests
└── README.md
```

Workspace `Cargo.toml`:

A virtual workspace has no `[package]`, so it does not inherit the edition's resolver: set
`resolver = "3"` explicitly (the edition-2024 default, MSRV-aware).

```toml
[workspace]
resolver = "3"
members = ["crates/*"]

[workspace.package]
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
license = "MIT OR Apache-2.0"
repository = "https://github.com/user/project"

[workspace.dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
```

## Rules

- Keep `src/main.rs` minimal; put logic in `src/lib.rs` or modules
- Use `src/lib.rs` for re-exports; define types in submodules
- Use `mod.rs` or `name.rs` consistently (prefer `name.rs` for flat modules)
- Integration tests in `tests/` test the public API only
- Examples should be runnable: `cargo run --example name`
- Use feature flags for optional functionality

## Recommended Starting Structure

### Binary Project

```
project/
├── Cargo.toml
├── src/
│   ├── main.rs         # Entry point, minimal
│   ├── lib.rs          # Core logic
│   ├── config.rs       # Configuration
│   ├── error.rs        # Error types
│   └── cli.rs          # CLI parsing
├── tests/
│   └── integration.rs
└── README.md
```

### Library Project

```
project/
├── Cargo.toml
├── src/
│   ├── lib.rs          # Public API
│   ├── error.rs        # Error types
│   └── types.rs        # Core types
├── tests/
│   └── integration.rs
├── examples/
│   └── basic.rs
└── README.md
```

## Cargo.toml Best Practices

```toml
[package]
name = "my-project"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"              # Minimum supported version
description = "A brief description"
license = "MIT OR Apache-2.0"
repository = "https://github.com/user/project"
keywords = ["keyword1", "keyword2"]
categories = ["development-tools"]

[dependencies]
# Group by purpose, alphabetize within groups

[dev-dependencies]
# Test-only dependencies

[features]
default = []
full = ["feature1", "feature2"]

[profile.release]
lto = true
strip = true
```

Add directories as needed, not preemptively.
