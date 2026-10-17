---
description: 'Rust test-driven development: Red-Green-Refactor, #[cfg(test)] placement, integration tests, mocking through traits. Use when writing tests or restructuring a test module.'
name: rust-tdd
---

# Test-Driven Development (TDD)

Work Red-Green-Refactor. Write one failing test for the next behavior and confirm it fails for
the right reason (an assertion, not a compile error in unrelated code). Add the minimum code to
pass, then refactor while green. The `rust-code-style` quality gate (fmt, clippy, test) must pass
before every commit.

## Conventions

- Names: `test_<function>_<scenario>_<expected>`, one behavior per test.
- Unit tests live in a `#[cfg(test)] mod tests { use super::*; … }` at the bottom of the same
  file. Integration tests in `tests/` exercise the public API only. Shared helpers go in
  `tests/common/mod.rs`, which Cargo does not build as a test binary.
- Async tests use `#[tokio::test]`. Parameterized cases use `rstest` (`#[rstest]` +
  `#[case(...)]`) or `test-case`. Panics use `#[should_panic(expected = "...")]`.
- Aim for meaningful coverage, not 100%. Under the `dev-team` plugin, `code-writer` enforces a
  ≥70% line-coverage floor on changed files. Measure it with `cargo llvm-cov` (check
  `cargo llvm-cov --version` first, and ask before installing `cargo-llvm-cov`).

## Mocking Through Traits

Put external dependencies behind a small trait and pass a hand-written fake in tests. Record
calls in a `Mutex`, not a `RefCell`, when the trait requires `Send + Sync`, because `RefCell` is
not `Sync`:

```rust
#[derive(Default)]
struct MockEmailSender {
    sent: Mutex<Vec<String>>,
}

impl EmailSender for MockEmailSender {
    fn send(&self, to: &str, _subject: &str, _body: &str) -> Result<(), Error> {
        self.sent.lock().unwrap().push(to.to_owned());
        Ok(())
    }
}

#[test]
fn test_signup_new_user_sends_welcome_mail() {
    let mailer = MockEmailSender::default();
    signup(&mailer, "alice@example.com").unwrap();
    assert_eq!(*mailer.sent.lock().unwrap(), ["alice@example.com"]);
}
```

For larger traits, `mockall` (`#[automock]`) generates the double. Keep hand-written fakes for
small ones.

## Integration Tests and External Services

- Tests that need a server start one in-process on an ephemeral port
  (`TcpListener::bind("127.0.0.1:0")`) rather than relying on one that happens to be running.
- Tests that need real external services (a database, a network API) stay out of the default
  `cargo test` gate. Mark them `#[ignore = "needs Postgres"]` and run them explicitly with
  `cargo test -- --ignored`.
