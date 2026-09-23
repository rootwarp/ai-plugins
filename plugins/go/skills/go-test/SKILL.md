---
description: 'Run Go tests with coverage via go test. Use when asked to run or debug Go tests.'
name: go-test
---

# /go-test

Run tests for the Go project.

## Instructions

Run tests with the race detector and coverage:

```bash
go test -race -cover ./...
```

Add `-v` for per-test output, `-run '^TestName$' ./path/to/pkg` for one test, and
`-coverprofile=coverage.out` followed by `go tool cover -func=coverage.out` for a per-function
coverage report.

If tests fail, analyze the failures and suggest fixes.
