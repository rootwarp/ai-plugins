---
description: 'Go test-driven development: Red-Green-Refactor, table-driven tests, test placement, interface fakes. Use when you write tests or restructure a test file.'
name: go-tdd
---

# Test-Driven Development

Use the Red, Green, Refactor cycle:

1. Red: Write a test for the next behavior. Make sure that it fails.
2. Green: Write the minimum code that makes the test pass.
3. Refactor: Clean up the code while the tests pass.

Test one behavior in each test or subtest.

## Conventions

- **Names:** Give a test the name of the unit (`TestParseSize`). Describe each scenario in the
  name of its subtest. A standalone test with one scenario can use
  `TestFunctionName_Scenario_ExpectedBehavior`.
- **Placement:** Put `*_test.go` files next to the code. Use the same package for unit tests.
  Use `package foo_test` to test only the exported API.
- **Structure:** Use table-driven tests with `t.Run` subtests for related cases.
- **Helpers:** Use `t.Helper()`, `t.Cleanup`, `t.TempDir()`, and `testdata/` for fixtures. Use
  `t.Context()` (Go 1.24+), not `context.Background()`. Add `t.Parallel()` only if the test
  shares no state.
- **Coverage:** Write tests that give meaningful coverage. Do not try for 100%. With the
  `dev-team` plugin, the pre-review gate requires ≥ 70% line coverage on changed files.
- Before you commit, run `go test -race ./...` (see the `go-test` skill).

## Fakes

Fake external dependencies through a small interface that the consumer declares. Use a fake that
you write manually, not a mocking framework. A manual fake is plain Go, and a refactor does not
break it. It also records exactly the data that the test asserts on.

```go
// service.go: the consumer declares what it needs
type Mailer interface {
    Send(ctx context.Context, m Mail) error
}

// service_test.go
type fakeMailer struct {
    sent []Mail
    err  error
}

func (f *fakeMailer) Send(_ context.Context, m Mail) error {
    f.sent = append(f.sent, m)
    return f.err
}
```
