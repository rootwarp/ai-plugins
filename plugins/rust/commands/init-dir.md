---
description: Initialize a Rust project (binary, library, or workspace) with Cargo, rustfmt, clippy, and cargo-deny config
---

# /rust:init-dir

Initialize a standard Rust project with Cargo configuration and common setup.

## Instructions

### Step 1: Gather Information

Ask the user:

1. **Project type:**
   - Binary (recommended) - executable application
   - Library - reusable library crate
   - Workspace - multi-crate project

2. **Project name** (used for package name and directory)

3. **Additional features:**
   - Async runtime (tokio/smol/none)
   - Serialization (serde)
   - CLI parsing (clap)
   - Error handling (`thiserror` for libraries; `anyhow` for binaries)

4. **License** (default `MIT OR Apache-2.0`) and **repository URL** (omit the `repository`
   field if the user has none; never write a placeholder URL)

5. **Optional files** — create only the ones the user selects:
   - `rustfmt.toml`, `rust-toolchain.toml`, `deny.toml`, `Makefile`

### Step 2: Check for an Existing Project

Before creating anything, check the target directory:

- If `<project-name>/Cargo.toml` already exists, stop and ask the user how to proceed; do not
  overwrite an existing manifest or sources
- If the directory exists and is not empty, list its contents and ask before continuing

If any `cargo` command below fails, stop, show the error, and ask the user; do not retry with
`--force`-style workarounds or delete files to make it pass.

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

Add `tests/`, `examples/`, and `benches/` when their first file exists — not preemptively (see
`rust-project-layout`).

**Workspace Project:** create the root manifest *before* the members, so `cargo new` fills in
`version.workspace = true` (and the other inherited keys) and skips creating nested git repos:

```bash
mkdir <project-name>
cd <project-name>
git init
```

Write the root `Cargo.toml` (virtual workspace; `resolver = "3"` must be explicit because there
is no `[package]` to infer it from):

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

Then create the members. Prefix package names with the project name (a bare `core` clashes with
Rust's built-in crate):

```bash
cargo new --lib --vcs none crates/core --name <project-name>-core
cargo new --vcs none crates/cli --name <project-name>-cli
```

### Step 4: Configure Cargo.toml

**Single crate:** edit the `Cargo.toml` that `cargo init` generated (keep its `name`, `version`,
and `edition`); add these keys under `[package]`, plus the release profile:

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

Use `cargo add` so versions resolve to the latest compatible release; do not hand-write version
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

**Workspace:** add each shared dependency to `[workspace.dependencies]` in the root
`Cargo.toml` (e.g. `serde = { version = "1", features = ["derive"] }`), then run
`cargo add <dep> -p <member>`; Cargo writes `<dep>.workspace = true` in the member.

### Step 6: Create Source Files

`<crate_ident>` below is the package name with `-` replaced by `_`.

**Binary** — keep `main.rs` minimal and put logic in `lib.rs` (see `rust-project-layout`):

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

Replace the `.gitignore` that `cargo init` generated (for a workspace, create it at the root):

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

Note: Commit `Cargo.lock` (current Cargo guidance for both binaries and libraries). Add it to
`.gitignore` only if a library explicitly prefers not to track it.

### Step 8: Write rustfmt.toml (optional)

Only if selected. `cargo fmt` already reads the edition from `Cargo.toml` and defaults to
`max_width = 100`; this file makes direct `rustfmt` runs (editors, pre-commit hooks) agree:

```toml
edition = "2024"
```

### Step 9: Write clippy.toml (only for non-default settings)

Skip this unless the user wants a non-default lint setting; Clippy already reads the MSRV from
`rust-version`. Example:

```toml
too-many-arguments-threshold = 5   # default is 7
```

### Step 10: Write rust-toolchain.toml (optional)

Only if selected. Pins the toolchain channel for everyone on the project:

```toml
[toolchain]
channel = "stable"
components = ["rustfmt", "clippy"]
```

### Step 11: Write deny.toml (optional)

Only if selected. Check `cargo deny --version`. If `cargo-deny` is installed, prefer
`cargo deny init` for the full commented template. If it is missing, write the minimal config
below and tell the user it needs `cargo install --locked cargo-deny`; do not install it
yourself (see the `rust-security` skill for the v2 format):

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

Show the created structure using `tree` or `ls -la` and confirm completion.

Run the quality gate from the `rust-code-style` skill:

```bash
cargo fmt --all
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace --all-features
```

If `deny.toml` was created and `cargo-deny` is installed, also run `cargo deny check`.

## Rules

- Use `cargo init` / `cargo new` as the starting point; never overwrite an existing project
- Follow standard Cargo project layout
- Commit `Cargo.lock` (binaries and libraries alike, per current Cargo guidance)
- Use edition 2024
- Add `rust-version` to specify MSRV
- Ask before installing any tool
