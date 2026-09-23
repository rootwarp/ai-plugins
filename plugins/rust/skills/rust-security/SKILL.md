---
description: 'Rust security: unsafe code guidelines, dependency auditing with cargo deny, input validation. Use when writing unsafe, handling untrusted input or secrets, or adding dependencies.'
name: rust-security
---

# Security

- Never hardcode secrets, credentials, or API keys
- Use environment variables or secure vaults for configuration
- Validate and sanitize ALL user input; use strong types
- Use parameterized queries with `sqlx` or similar to prevent SQL injection
- Escape output properly to prevent XSS in web applications
- Generate keys and tokens from a CSPRNG: `getrandom::fill`, `rand::rngs::OsRng` (via
  `TryRngCore` in rand 0.9), or `rand::rng()` (a reseeding ChaCha CSPRNG); never a
  deterministically seeded RNG (`StdRng::seed_from_u64`) or a non-crypto one (`SmallRng`)
- Set appropriate timeouts on network operations
- Use TLS for all network communication (`rustls` or `native-tls`)
- Minimize use of `unsafe`; document safety invariants when required
- Run `cargo clippy` to catch potential issues
- Keep dependencies updated; audit with `cargo deny`
- Follow principle of least privilege for file permissions
- Log security events but never log sensitive data
- Use constant-time comparison for secrets (`subtle` crate)
- Avoid deserializing untrusted data without validation
- Use `secrecy` crate for sensitive values in memory

## Unsafe Code Guidelines

When `unsafe` is necessary:

```rust
/// Converts a byte slice to a string without UTF-8 validation.
///
/// # Safety
///
/// The caller must ensure that `bytes` contains valid UTF-8.
pub unsafe fn str_from_utf8_unchecked(bytes: &[u8]) -> &str {
    // SAFETY: Caller guarantees bytes is valid UTF-8
    unsafe { std::str::from_utf8_unchecked(bytes) }
}
```

Edition 2024 warns on unsafe operations in an `unsafe fn` body that are not wrapped in an
`unsafe {}` block (`unsafe_op_in_unsafe_fn`), and the clippy gate turns that warning into an
error — keep the inner block.

- Always document `# Safety` requirements
- Add `// SAFETY:` comments explaining why invariants are upheld
- Prefer safe abstractions that encapsulate unsafe code
- Audit all `unsafe` blocks during code review

## Dependency Security

First check that the tool is installed:

```bash
cargo deny --version
```

If it is missing, stop and ask the user before installing anything; suggest
`cargo install --locked cargo-deny` (or `cargo binstall cargo-deny` for a prebuilt binary). Do
not install it on your own.

```bash
# Generate a full, commented deny.toml template (only if none exists; do not overwrite)
cargo deny init

# Check security advisories, licenses, bans, and sources
cargo deny check
```

Example `deny.toml` (cargo-deny v2 config — the old `vulnerability`/`notice` keys were removed;
matching advisories are errors unless listed in `ignore`):

```toml
[advisories]
yanked = "warn"
# ignore = [{ id = "RUSTSEC-0000-0000", reason = "no patched release yet; tracked in #123" }]

[licenses]
allow = ["MIT", "Apache-2.0", "BSD-3-Clause", "Unicode-3.0"]

[bans]
multiple-versions = "warn"
```
