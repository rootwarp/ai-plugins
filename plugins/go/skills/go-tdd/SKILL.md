---
description: 'Go test-driven development: Red-Green-Refactor, table-driven tests, test placement, interface fakes. Use when writing tests or restructuring a test file.'
name: go-tdd
---

# Test-Driven Development (TDD)

- Write tests BEFORE implementing functionality
- Follow the Red-Green-Refactor cycle:
  1. Write a failing test that defines expected behavior
  2. Write minimal code to make the test pass
  3. Refactor while keeping tests green
- Each test should test ONE behavior
- Name a test after the unit (`TestParseSize`) and describe each scenario in its subtest name;
  a standalone test for one scenario may use `TestFunctionName_Scenario_ExpectedBehavior`
- Place tests in `*_test.go` files next to the code; use the same package for unit tests and
  `package foo_test` when you want to test only the exported API
- Use table-driven tests with subtests (`t.Run`) for related cases
- Fake external dependencies through small consumer-side interfaces; prefer a hand-written fake
  over a mocking framework
- Mark helpers with `t.Helper()`; clean up with `t.Cleanup`; use `t.TempDir()` and `testdata/`
  for files; use `t.Context()` (Go 1.24+) instead of `context.Background()`; add `t.Parallel()`
  only when the test shares no state
- Aim for meaningful coverage, not 100% coverage
- When using the `dev-team` plugin, `code-writer` enforces a ≥70% coverage floor for new code
- Run tests before committing: `go test -race ./...`

## Table-Driven Test

```go
func TestParseSize(t *testing.T) {
    tests := []struct {
        name    string
        in      string
        want    int64
        wantErr bool
    }{
        {name: "plain bytes", in: "42", want: 42},
        {name: "kilobytes", in: "2KB", want: 2048},
        {name: "unknown unit", in: "2XB", wantErr: true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseSize(tt.in)
            if (err != nil) != tt.wantErr {
                t.Fatalf("ParseSize(%q) error = %v, wantErr %v", tt.in, err, tt.wantErr)
            }
            if tt.wantErr {
                return
            }
            if got != tt.want {
                t.Errorf("ParseSize(%q) = %d, want %d", tt.in, got, tt.want)
            }
        })
    }
}
```

## Fake Through a Consumer-Side Interface

```go
// service.go — the consumer declares what it needs
type Mail struct {
    To, Subject, Body string
}

type Mailer interface {
    Send(ctx context.Context, m Mail) error
}

// service_test.go
type fakeMailer struct {
    sent []string
    err  error
}

func (f *fakeMailer) Send(_ context.Context, m Mail) error {
    f.sent = append(f.sent, m.To)
    return f.err
}

func TestSignup_NewUser_SendsWelcomeMail(t *testing.T) {
    mailer := &fakeMailer{}
    svc := NewSignupService(mailer)

    if err := svc.Signup(t.Context(), "alice@example.com"); err != nil {
        t.Fatalf("Signup() error = %v", err)
    }
    if len(mailer.sent) != 1 || mailer.sent[0] != "alice@example.com" {
        t.Errorf("sent = %v, want one mail to alice@example.com", mailer.sent)
    }
}
```
