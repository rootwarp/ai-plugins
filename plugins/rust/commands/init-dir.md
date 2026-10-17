---
description: Initialize a Rust project (binary, library, or workspace) with Cargo, rustfmt, clippy, and cargo-deny config
---

# /rust:init-dir

Initialize a Rust project with Cargo configuration and common setup.

## Instructions

### Step 1: Gather Information

Ask the user for whatever the request did not already answer:

1. **Project type:** Binary (recommended), Library, or Workspace (multi-crate)
2. **Project name** (package name and directory)
3. **Dependencies:** async runtime (tokio / smol / none), serde, clap, and error handling
   (`thiserror` for libraries, `anyhow` for binaries)
4. **License** (default `MIT OR Apache-2.0`) and **repository URL**. With no URL, omit the
   `repository` field rather than writing a placeholder.
5. **Optional files:** `rustfmt.toml`, `rust-toolchain.toml`, `deny.toml`, `Makefile`. Create
   only the ones selected.

### Step 2: Check for an Existing Project

- `<project-name>/Cargo.toml` already exists → stop and ask how to proceed. Never overwrite an
  existing manifest or sources.
- The directory exists and is not empty → list its contents and ask before continuing.

If any `cargo` command below fails, stop, show the error, and ask the user. Don't work around
it with `--force` or by deleting files.

### Step 3: Initialize with Cargo

`cargo init` creates `src/`, a starter `.gitignore`, and a git repository (unless the directory
is already inside one).

**Binary Project:**

```bash
cargo init <project-name>
cd <project-name>
```

**Library Project:**

```bash
cargo init --lib <project-name>
cd <project-name>
```

Add `tests/`, `examples/`, and `benches/` when their first file exists (see
`rust-project-layout`).

**Workspace Project:** write the root manifest before creating members, so `cargo new` fills in
`version.workspace = true` and the other inherited keys:

```bash
mkdir <project-name>
cd <project-name>
git init
```

Root `Cargo.toml` (a virtual workspace has no `[package]` to infer the resolver from, so
`resolver = "3"` is explicit):

```toml
[workspace]
resolver = "3"
members = ["crates/*"]

[workspace.package]
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
license = "<license>"
repository = "<repository-url>"   # omit if none

[workspace.dependencies]
# Shared dependency versions; members opt in with `<dep>.workspace = true`
```

Create the members, prefixing package names with the project name (a bare `core` clashes with
the built-in crate):

```bash
cargo new --lib --vcs none crates/core --name <project-name>-core
cargo new --vcs none crates/cli --name <project-name>-cli
```

### Step 4: Configure Cargo.toml

**Single crate:** keep the generated `name`, `version`, and `edition`; add these keys under
`[package]`, plus the release profile:

```toml
rust-version = "1.85"
description = "A brief description"
license = "<license>"
repository = "<repository-url>"   # omit if none

[profile.release]
lto = true
strip = true
```

**Workspace:** members already inherit `version`, `edition`, `rust-version`, `license`, and
`repository` from `[workspace.package]`; add `description` per member.

### Step 5: Add Selected Dependencies

Use `cargo add` so versions resolve to the latest compatible release rather than hand-written
numbers.

**Single crate:**

```bash
# Async (tokio): enable only what is used; `full` only for a quick prototype binary
cargo add tokio --features rt-multi-thread,macros
# Async (smol)
cargo add smol

# Serialization
cargo add serde --features derive
cargo add serde_json

# CLI (binaries)
cargo add clap --features derive

# Errors: thiserror in libraries, anyhow in binaries
cargo add thiserror
cargo add anyhow
```

**Workspace:** add each shared dependency to the root `[workspace.dependencies]` (e.g.
`serde = { version = "1", features = ["derive"] }`), then run `cargo add <dep> -p <member>`,
which writes `<dep>.workspace = true` in the member.

### Step 6: Create Source Files

`<crate_ident>` below is the package name with `-` replaced by `_`.

**Binary:** keep `main.rs` minimal and put the logic in `lib.rs`.

`src/main.rs`:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    <crate_ident>::run()
}
```

`src/lib.rs`:

```rust
//! <project-name>: a brief description.

/// Runs the application.
///
/// # Errors
///
/// Returns an error if the application fails to start.
pub fn run() -> Result<(), Box<dyn std::error::Error>> {
    println!("Hello, world!");
    Ok(())
}
```

(With `anyhow` selected, return `anyhow::Result<()>` from both instead.)

**Library** (`src/lib.rs`):

```rust
//! <project-name>: a brief description of the library.

/// Adds two numbers.
///
/// # Examples
///
/// ```
/// assert_eq!(<crate_ident>::add(2, 2), 4);
/// ```
pub fn add(left: u64, right: u64) -> u64 {
    left + right
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add_two_positive_numbers_returns_sum() {
        assert_eq!(add(2, 2), 4);
    }
}
```

### Step 7: Write .gitignore

Replace the generated `.gitignore` (for a workspace, create it at the root):

```gitignore
# Generated by Cargo
/target/

# IDE
.idea/
.vscode/
*.swp
*.swo
*~

# OS files
.DS_Store
Thumbs.db

# Environment files
.env
.env.local
*.env

# Build artifacts
*.so
*.dylib
*.dll

# Coverage
*.profraw
*.profdata
coverage/
lcov.info
tarpaulin-report.html
```

`Cargo.lock` stays tracked, for binaries and libraries alike. Ignore it only if the user
explicitly asks to for a library.

### Step 8: Write rustfmt.toml (optional)

Only if selected. `cargo fmt` already reads the edition from `Cargo.toml`. This file makes
direct `rustfmt` runs (editors, pre-commit hooks) agree:

```toml
edition = "2024"
```

### Step 9: Write clippy.toml (only for non-default settings)

Only for a non-default lint setting the user wants; Clippy already reads the MSRV from
`rust-version`. Example:

```toml
too-many-arguments-threshold = 5   # default is 7
```

### Step 10: Write rust-toolchain.toml (optional)

Only if selected. It pins the toolchain for everyone on the project:

```toml
[toolchain]
channel = "stable"
components = ["rustfmt", "clippy"]
```

### Step 11: Write deny.toml (optional)

Only if selected. If `cargo deny --version` succeeds, prefer `cargo deny init` for the full
commented template. Otherwise write the minimal v2 config below and tell the user it needs
`cargo install --locked cargo-deny`; don't install it yourself (see `rust-security`):

```toml
[advisories]
yanked = "warn"

[licenses]
allow = ["MIT", "Apache-2.0", "BSD-3-Clause", "Unicode-3.0"]

[bans]
multiple-versions = "warn"
```

### Step 12: Create Makefile (optional)

Only if selected:

```makefile
.PHONY: all build build-release run test test-verbose lint fmt fmt-check audit clean doc help

all: build

build: ## Build the project
	cargo build

build-release: ## Build release version
	cargo build --release

run: ## Run the application
	cargo run

test: ## Run tests
	cargo test --workspace --all-features

test-verbose: ## Run tests with output
	cargo test --workspace --all-features -- --nocapture

lint: ## Run clippy
	cargo clippy --workspace --all-targets --all-features -- -D warnings

fmt: ## Format code
	cargo fmt --all

fmt-check: ## Check formatting
	cargo fmt --all --check

audit: ## Check advisories, licenses, bans, and sources (requires cargo-deny)
	cargo deny check

clean: ## Clean build artifacts
	cargo clean

doc: ## Generate documentation (add --open to view in a browser)
	cargo doc --no-deps

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-15s\033[0m %s\n", $$1, $$2}'
```

Omit the `run` target for a library.

### Step 13: Display Result

Show the created structure (`tree` or `ls -la`), then run the `rust-code-style` quality gate:

```bash
cargo fmt --all
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace --all-features
```

If `deny.toml` was created and `cargo-deny` is installed, also run `cargo deny check`.

## Rules

- Start from `cargo init` / `cargo new`, and never overwrite an existing project.
- Edition 2024 with `rust-version` set. Commit `Cargo.lock`.
- Ask before installing any tool.
