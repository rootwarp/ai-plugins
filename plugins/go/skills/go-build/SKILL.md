---
description: 'Build the Go project with go build and analyze any build errors. Use when asked to build or compile Go code.'
name: go-build
allowed-tools: Bash(go build:*), Bash(go vet:*), Read
---

# /go:go-build

Build the Go project.

## Instructions

Run the Go build command to compile the project:

```bash
go build ./...
```

`go build` skips `_test.go` files; run `go vet ./...` as well when test code changed, since it
type-checks tests too.

If there are build errors, analyze them and suggest fixes.
