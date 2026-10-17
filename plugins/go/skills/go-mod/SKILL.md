---
description: 'Manage Go module dependencies: go mod tidy, go get, go mod why. Use after adding or removing imports, or when adding, upgrading, or removing a dependency.'
name: go-mod
allowed-tools: Bash(go mod tidy:*), Bash(go mod why:*), Bash(go list -m:*), Bash(git diff:*), Read
---

# /go:go-mod

Manage Go modules. After imports change, run:

```bash
go mod tidy
```

It needs network access (or a populated module cache) and can rewrite the `go` and `toolchain`
lines. Review `git diff go.mod go.sum` afterwards and point out any directive change to the user.

| Task | Command |
|------|---------|
| Add or pin a dependency | `go get example.com/pkg@v1.2.3` |
| Upgrade one dependency | `go get example.com/pkg@latest` |
| Remove a dependency | delete its imports, then `go mod tidy` (or `go get example.com/pkg@none`) |
| Why is it required? | `go mod why -m example.com/pkg` |
| Current versions | `go list -m all` |
| Track a dev tool (Go 1.24+) | `go get -tool example.com/cmd/tool@version` |

Ask before a broad upgrade (`go get -u ./...`), since it can pull in many unrelated changes.

In a workspace (`go.work` present), run tidy inside each module directory. `go work sync` pushes
the workspace's resolved versions back into the member modules.
