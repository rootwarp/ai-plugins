---
description: Initialize a Rust project (binary, library, or workspace) with Cargo, rustfmt, clippy, and cargo-deny configuration
---

# /rust:init-dir

Initialize a Rust project with a Cargo configuration and the usual setup files.

## Instructions

### Step 1: Get the Information

Ask the user for each item that the request does not give:

1. **Project type:** Binary (recommended), Library, or Workspace (multi-crate)
2. **Project name** (package name and directory)
3. **Dependencies:** async runtime (tokio / smol / none), serde, clap, and error handling
   (`thiserror` for libraries, `anyhow` for binaries)
4. **License** (default `MIT OR Apache-2.0`) and **repository URL**. If there is no URL, do
   not write the `repository` field. Do not write a placeholder.
5. **Optional files:** `rustfmt.toml`, `rust-toolchain.toml`, `deny.toml`, `Makefile`. Create
   only the files that the user selects.

### Step 2: Check if a Project Exists

- If `<project-name>/Cargo.toml` exists, stop and ask the user what to do. Never overwrite a
  manifest or source files that exist.
- If the directory exists and is not empty, list its contents. Ask the user before you continue.

If a `cargo` command below fails, stop. Show the error and ask the user. Do not use `--force` or
delete files to go around the error.

### Step 3: Initialize with Cargo

`cargo init` creates `src/`, a starter `.gitignore`, and a git repository. It does not create
the repository if the directory is already in a git repository.

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

Add `tests/`, `examples/`, and `benches/` when you add their first file (see
`rust-project-layout`).

**Workspace Project:** Write the root manifest before you create the members. Then `cargo new`
adds `version.workspace = true` and the other inherited keys:

```bash
mkdir <project-name>
cd <project-name>
git init
```

Root `Cargo.toml`: A virtual workspace has no `[package]` that gives the resolver, so
`resolver = "3"` is explicit:

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

Create the members. Start each package name with the project name, because a package that has
only the name `core` has a conflict with the built-in crate:

```bash
cargo new --lib --vcs none crates/core --name <project-name>-core
cargo new --vcs none crates/cli --name <project-name>-cli
```

### Step 4: Configure Cargo.toml

**Single crate:** Keep the generated `name`, `version`, and `edition`. Add these keys under
`[package]`, and add the release profile:

```toml
rust-version = "1.85"
description = "A brief description"
license = "<license>"
repository = "<repository-url>"   # omit if none

[profile.release]
lto = true
strip = true
```

**Workspace:** The members already inherit `version`, `edition`, `rust-version`, `license`, and
`repository` from `[workspace.package]`. Add a `description` to each member.

### Step 5: Add Selected Dependencies

Use `cargo add`. It sets each version to the latest compatible release. Do not write version
numbers by hand.

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

**Workspace:** Add each shared dependency to the root `[workspace.dependencies]`, for example
`serde = { version = "1", features = ["derive"] }`. Then run `cargo add <dep> -p <member>`. This
command writes `<dep>.workspace = true` in the member.

### Step 6: Create Source Files

In the files below, `<crate_ident>` is the package name with each `-` changed to `_`.

**Binary:** Keep `main.rs` minimal. Put the logic in `lib.rs`.

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

If the user selects `anyhow`, return `anyhow::Result<()>` from both functions.

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

Replace the generated `.gitignore`. For a workspace, create it at the root:

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

Keep `Cargo.lock` tracked for binaries and also for libraries. Ignore it only if the user
explicitly asks for this for a library.

### Step 8: Write rustfmt.toml (optional)

Write this file only if the user selects it. `cargo fmt` already reads the edition from
`Cargo.toml`. This file gives the same edition to direct `rustfmt` runs (editors, pre-commit
hooks):

```toml
edition = "2024"
```

### Step 9: Write clippy.toml (only for non-default settings)

Write this file only for a non-default lint setting that the user wants. Clippy already reads
the MSRV from `rust-version`. Example:

```toml
too-many-arguments-threshold = 5   # default is 7
```

### Step 10: Write rust-toolchain.toml (optional)

Write this file only if the user selects it. It pins the toolchain for all people on the
project:

```toml
[toolchain]
channel = "stable"
components = ["rustfmt", "clippy"]
```

### Step 11: Write deny.toml (optional)

Write this file only if the user selects it. If `cargo deny --version` succeeds, use
`cargo deny init` to get the full template with comments. If not, write the minimal v2
configuration below. Tell the user that it needs `cargo install --locked cargo-deny`. Do not
install it yourself (see `rust-security`):

```toml
[advisories]
yanked = "warn"

[licenses]
allow = ["MIT", "Apache-2.0", "BSD-3-Clause", "Unicode-3.0"]

[bans]
multiple-versions = "warn"
```

### Step 12: Create Makefile (optional)

Create this file only if the user selects it:

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

For a library, do not write the `run` target.

### Step 13: Show the Result

Show the structure that you created (`tree` or `ls -la`). Then run the `rust-code-style`
quality gate:

```bash
cargo fmt --all
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace --all-features
```

If you created `deny.toml` and `cargo-deny` is installed, also run `cargo deny check`.

## Rules

- Start with `cargo init` or `cargo new`. Never overwrite a project that exists.
- Use edition 2024, and set `rust-version`. Commit `Cargo.lock`.
- Ask before you install a tool.
