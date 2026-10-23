---
description: 'Rust test-driven development: Red-Green-Refactor, #[cfg(test)] placement, integration tests, and mocks through traits. Use when you write tests or restructure a test module.'
name: rust-tdd
---

# Test-Driven Development (TDD)

Use the Red-Green-Refactor cycle:

1. Write one test for the next behavior. The test must fail.
2. Make sure that it fails for the correct reason: an assertion, not a compile error in
   unrelated code.
3. Add the minimum code that makes the test pass.
4. Refactor while the tests pass.

The `rust-code-style` quality gate (fmt, clippy, test) must pass before each commit.

## Conventions

- Names: Use `test_<function>_<scenario>_<expected>`. Test one behavior in each test.
- Put unit tests in a `#[cfg(test)] mod tests { use super::*; … }` at the bottom of the same
  file. Integration tests in `tests/` test only the public API. Put shared helpers in
  `tests/common/mod.rs`. Cargo does not build this file as a test binary.
- For async tests, use `#[tokio::test]`. For parameterized cases, use `rstest` (`#[rstest]` +
  `#[case(...)]`) or `test-case`. For panics, use `#[should_panic(expected = "...")]`.
- Get useful coverage. Do not try to get 100%. In the `dev-team` plugin, `code-writer` requires
  a minimum line coverage of 70% (≥70%) on changed files. Use `cargo llvm-cov` to measure it.
  First run `cargo llvm-cov --version`. Ask before you install `cargo-llvm-cov`.

## Mocks Through Traits

Put external dependencies behind a small trait. In tests, pass a hand-written fake. If the
trait requires `Send + Sync`, record calls in a `Mutex`, not a `RefCell`, because `RefCell` is
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

For larger traits, `mockall` (`#[automock]`) generates the test double. For small traits, use
hand-written fakes.

## Integration Tests and External Services

- If a test needs a server, it starts the server in-process on an ephemeral port
  (`TcpListener::bind("127.0.0.1:0")`). Do not use a server that possibly runs already.
- Keep tests that need real external services (a database, a network API) out of the default
  `cargo test` gate. Mark them `#[ignore = "needs Postgres"]`. Run them explicitly with
  `cargo test -- --ignored`.
