---
description: 'Build the Go project with go build and analyze any build errors. Use when asked to build or compile Go code.'
name: go-build
allowed-tools: Bash(go build:*), Bash(go vet:*), Read
---

# /go:go-build

Build the Go project:

```bash
go build ./...
```

`go build` skips `_test.go` files. When test code changed, also run `go vet ./...`, which
type-checks tests too.

On errors, find the root cause in the source and propose the fix.
