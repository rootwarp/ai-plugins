---
description: 'Rust security: unsafe code guidelines, dependency audits with cargo deny, and input validation. Use when you write unsafe code, handle untrusted input or secrets, or add dependencies.'
name: rust-security
---

# Security

## Input, Secrets, and Crypto

- At the boundary, parse untrusted input into strong types (newtypes with constructors that
  validate). Set size limits before you deserialize. Do not deserialize untrusted data directly
  into internal types.
- Send SQL through bound parameters (`sqlx::query!` or `.bind()`). Never use `format!`.
- Get secrets from the environment or a vault, never from the source code. Wrap them in
  `secrecy::SecretString` (or `SecretBox`). Then they do not appear in `Debug` output and logs.
  Compare secrets with the constant-time `ct_eq` of `subtle`.
- Get keys and tokens from a CSPRNG: `getrandom::fill`, `rand::rngs::OsRng` (through
  `TryRngCore` in rand 0.9), or `rand::rng()` (a ChaCha CSPRNG that reseeds). Never use an RNG
  with a deterministic seed (`StdRng::seed_from_u64`) or an RNG that is not for cryptography
  (`SmallRng`).
- Use `rustls` (or `native-tls`) for TLS. Put a timeout on each network operation.
- Log security events. Never log the sensitive data in them.

## Unsafe Code

Do not use `unsafe` unless a safe abstraction cannot do the job. Wrap the `unsafe` code in the
smallest possible safe API. Each `unsafe fn` documents its contract under `# Safety`. Each
`unsafe` block has a `// SAFETY:` comment that tells why the invariants are true:

```rust
/// # Safety
///
/// The caller must ensure that `bytes` contains valid UTF-8.
pub unsafe fn str_from_utf8_unchecked(bytes: &[u8]) -> &str {
    // SAFETY: the caller guarantees `bytes` is valid UTF-8.
    unsafe { std::str::from_utf8_unchecked(bytes) }
}
```

Edition 2024 gives a warning when the body of an `unsafe fn` does an unsafe operation outside an
`unsafe {}` block (`unsafe_op_in_unsafe_fn`). The clippy gate changes that warning into an error.
Thus, keep the inner block. Reviewers audit each `unsafe` block.

## Dependency Audits

Run `cargo deny --version` to find if the tool is installed. If it is not installed, ask the
user before you install anything. Recommend `cargo install --locked cargo-deny` (or
`cargo binstall cargo-deny`).

```bash
cargo deny init    # full commented deny.toml, only when none exists (never overwrite)
cargo deny check   # advisories, licenses, bans, sources
```

This is a minimal `deny.toml` in the cargo-deny v2 format. The v2 format removed the old
`vulnerability` and `notice` keys. An advisory that matches is an error unless `ignore` lists it:

```toml
[advisories]
yanked = "warn"
# ignore = [{ id = "RUSTSEC-0000-0000", reason = "no patched release yet; tracked in #123" }]

[licenses]
allow = ["MIT", "Apache-2.0", "BSD-3-Clause", "Unicode-3.0"]

[bans]
multiple-versions = "warn"
```
