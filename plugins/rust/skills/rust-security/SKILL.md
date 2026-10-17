---
description: 'Rust security: unsafe code guidelines, dependency auditing with cargo deny, input validation. Use when writing unsafe, handling untrusted input or secrets, or adding dependencies.'
name: rust-security
---

# Security

## Input, Secrets, and Crypto

- Parse untrusted input into strong types at the boundary (newtypes whose constructors validate).
  Bound sizes before deserializing, and don't deserialize untrusted data straight into internal
  types.
- SQL goes through bound parameters (`sqlx::query!` / `.bind()`), never `format!`.
- Secrets come from the environment or a vault, never source. Wrap them in `secrecy::SecretString`
  (or `SecretBox`) so they stay out of `Debug` output and logs. Compare them with `subtle`'s
  constant-time `ct_eq`.
- Keys and tokens come from a CSPRNG: `getrandom::fill`, `rand::rngs::OsRng` (via `TryRngCore`
  in rand 0.9), or `rand::rng()` (a reseeding ChaCha CSPRNG). Never use a deterministically
  seeded RNG (`StdRng::seed_from_u64`) or a non-crypto one (`SmallRng`).
- TLS via `rustls` (or `native-tls`). Put timeouts on every network operation.
- Log security events, never the sensitive data in them.

## Unsafe Code

Avoid `unsafe` unless a safe abstraction cannot do the job, and wrap it in the smallest safe API
you can. Every `unsafe fn` documents its contract under `# Safety`, and every `unsafe` block gets
a `// SAFETY:` comment saying why the invariants hold:

```rust
/// # Safety
///
/// The caller must ensure that `bytes` contains valid UTF-8.
pub unsafe fn str_from_utf8_unchecked(bytes: &[u8]) -> &str {
    // SAFETY: the caller guarantees `bytes` is valid UTF-8.
    unsafe { std::str::from_utf8_unchecked(bytes) }
}
```

Edition 2024 warns when an `unsafe fn` body performs an unsafe operation outside an `unsafe {}`
block (`unsafe_op_in_unsafe_fn`), and the clippy gate turns that warning into an error, so keep
the inner block. Reviewers audit every `unsafe` block.

## Dependency Auditing

Check for the tool with `cargo deny --version`. If it is missing, ask the user before installing
anything; suggest `cargo install --locked cargo-deny` (or `cargo binstall cargo-deny`).

```bash
cargo deny init    # full commented deny.toml, only when none exists (never overwrite)
cargo deny check   # advisories, licenses, bans, sources
```

Minimal `deny.toml` in the cargo-deny v2 format. The old `vulnerability`/`notice` keys are gone,
and matching advisories are errors unless listed in `ignore`:

```toml
[advisories]
yanked = "warn"
# ignore = [{ id = "RUSTSEC-0000-0000", reason = "no patched release yet; tracked in #123" }]

[licenses]
allow = ["MIT", "Apache-2.0", "BSD-3-Clause", "Unicode-3.0"]

[bans]
multiple-versions = "warn"
```
