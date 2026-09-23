---
description: 'Rust test-driven development: Red-Green-Refactor, #[cfg(test)] placement, integration tests, mocking through traits. Use when writing tests or restructuring a test module.'
name: rust-tdd
---

# Test-Driven Development (TDD)

- Write tests BEFORE implementing functionality
- Follow the Red-Green-Refactor cycle:
  1. Write a failing test that defines expected behavior
  2. Write minimal code to make the test pass
  3. Refactor while keeping tests green
- Each test should test ONE behavior
- Use descriptive test names: `test_function_scenario_expected_behavior`
- Place unit tests in `#[cfg(test)]` module in the same file
- Use `tests/` directory for integration tests
- Mock external dependencies using traits and test doubles
- Aim for meaningful coverage, not 100% coverage
- When using the `dev-team` plugin, `code-writer` enforces a ≥70% coverage floor for new code
- See the `rust-code-style` skill for the MANDATORY fmt/clippy/test gate that must pass before
  every commit

## Test Organization

```rust
// src/user.rs
pub struct User {
    pub name: String,
    pub email: String,
}

impl User {
    pub fn new(name: &str, email: &str) -> Result<Self, ValidationError> {
        if name.is_empty() {
            return Err(ValidationError::EmptyName);
        }
        Ok(Self {
            name: name.to_string(),
            email: email.to_string(),
        })
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_new_user_valid_input_creates_user() {
        let user = User::new("Alice", "alice@example.com").unwrap();
        assert_eq!(user.name, "Alice");
        assert_eq!(user.email, "alice@example.com");
    }

    #[test]
    fn test_new_user_empty_name_returns_error() {
        let result = User::new("", "alice@example.com");
        assert!(matches!(result, Err(ValidationError::EmptyName)));
    }
}
```

## Test Utilities

- Use `#[test]` for synchronous tests
- Use `#[tokio::test]` for async tests
- Use `#[should_panic(expected = "message")]` for panic tests
- Use `assert!`, `assert_eq!`, `assert_ne!` for assertions
- Use `rstest` or `test-case` for parameterized tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use rstest::rstest;

    #[rstest]
    #[case(2, 2, 4)]
    #[case(0, 5, 5)]
    #[case(-1, 1, 0)]
    fn test_add(#[case] a: i32, #[case] b: i32, #[case] expected: i32) {
        assert_eq!(add(a, b), expected);
    }
}
```

## Mocking with Traits

```rust
// Define trait for external dependency
trait EmailSender: Send + Sync {
    fn send(&self, to: &str, subject: &str, body: &str) -> Result<(), Error>;
}

// Production implementation
struct SmtpEmailSender { /* ... */ }
impl EmailSender for SmtpEmailSender { /* ... */ }

// Test mock
#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::Mutex;

    // `Mutex`, not `RefCell`: the trait requires `Send + Sync`, and `RefCell` is not `Sync`.
    #[derive(Default)]
    struct MockEmailSender {
        sent: Mutex<Vec<(String, String, String)>>,
    }

    impl EmailSender for MockEmailSender {
        fn send(&self, to: &str, subject: &str, body: &str) -> Result<(), Error> {
            self.sent.lock().unwrap().push((to.into(), subject.into(), body.into()));
            Ok(())
        }
    }

    #[test]
    fn test_signup_new_user_sends_welcome_mail() {
        let mailer = MockEmailSender::default();
        signup(&mailer, "alice@example.com").unwrap();
        assert_eq!(mailer.sent.lock().unwrap().len(), 1);
    }
}
```

For larger traits, `mockall` (`#[automock]`) generates the double; keep hand-written fakes for
small ones.

## Integration Tests

Place in `tests/` directory:

```rust
// tests/api_integration.rs
use my_crate::Client;

#[test]
fn test_client_connection() {
    let client = Client::new("localhost:8080");
    assert!(client.ping().is_ok());
}
```
