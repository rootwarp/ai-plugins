---
description: 'Run Go tests with coverage via go test. Use when asked to run or debug Go tests.'
name: go-test
allowed-tools: Bash(go test:*), Bash(go tool cover:*), Read
---

# /go:go-test

Run tests for the Go project.

## Instructions

Run tests with the race detector and coverage:

```bash
go test -race -cover ./...
```

`-race` needs cgo and a C compiler. If it fails with a cgo error (e.g. `-race requires cgo`),
rerun without `-race` and tell the user the race detector was skipped.

Add `-v` for per-test output, `-run '^TestName$' ./path/to/pkg` for one test, and
`-coverprofile=coverage.out` followed by `go tool cover -func=coverage.out` for a per-function
coverage report.

If tests fail, analyze the failures and suggest fixes.
