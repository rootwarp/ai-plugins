---
description: 'Run Go tests with coverage through go test. Use when the user asks you to run or debug Go tests.'
name: go-test
allowed-tools: Bash(go test:*), Bash(go tool cover:*), Read
---

# /go:go-test

Run the tests with the race detector and coverage:

```bash
go test -race -cover ./...
```

`-race` needs cgo and a C compiler. If the command fails with a cgo error (for example
`-race requires cgo`), run it again without `-race`. Then tell the user that the race detector
did not run.

These variants are useful:

- `-v` shows the output of each test.
- `-run '^TestName$' ./path/to/pkg` runs one test.
- `-coverprofile=coverage.out`, then `go tool cover -func=coverage.out`, shows the coverage of
  each function.

If a test fails, find the root cause and propose a fix.
