---
description: 'Go test-driven development: Red-Green-Refactor, table-driven tests, test placement, interface fakes. Use when writing tests or restructuring a test file.'
name: go-tdd
---

# Test-Driven Development

Work Red → Green → Refactor: a failing test for the next behavior, the minimum code to pass it,
then clean up with the tests green. One behavior per test or subtest.

## Conventions

- **Naming:** name a test after the unit (`TestParseSize`) and describe each scenario in its
  subtest name. A standalone single-scenario test may use
  `TestFunctionName_Scenario_ExpectedBehavior`.
- **Placement:** `*_test.go` next to the code. Use the same package for unit tests and
  `package foo_test` to test only the exported API.
- **Structure:** table-driven tests with `t.Run` subtests for related cases.
- **Helpers:** `t.Helper()`, `t.Cleanup`, `t.TempDir()`, and `testdata/` for fixtures. Use
  `t.Context()` (Go 1.24+) instead of `context.Background()`. Add `t.Parallel()` only when the
  test shares no state.
- **Coverage:** aim for meaningful coverage, not 100%. Under the `dev-team` plugin, the
  pre-review gate requires ≥ 70% line coverage on changed files.
- Run `go test -race ./...` before committing (see the `go-test` skill).

## Fakes

Fake external dependencies through a small interface that the consumer declares. Prefer a
hand-written fake to a mocking framework: it is plain Go, it is refactor-safe, and it records
exactly what the test asserts on.

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
