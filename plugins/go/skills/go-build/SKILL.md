---
description: 'Build the Go project with go build and analyze the build errors. Use when the user asks you to build or compile Go code.'
name: go-build
allowed-tools: Bash(go build:*), Bash(go vet:*), Read
---

# /go:go-build

Build the Go project:

```bash
go build ./...
```

`go build` does not compile `_test.go` files. If test code changed, also run `go vet ./...`.
This command also does a type check of the tests.

If the build fails, find the root cause in the source and propose a fix.
