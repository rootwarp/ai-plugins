---
description: 'Manage Go module dependencies: go mod tidy, go get, go mod why. Use after you add or remove imports, or when you add, upgrade, or remove a dependency.'
name: go-mod
allowed-tools: Bash(go mod tidy:*), Bash(go mod why:*), Bash(go list -m:*), Bash(git diff:*), Read
---

# /go:go-mod

Manage Go modules. After the imports change, run this command:

```bash
go mod tidy
```

The command needs network access (or a populated module cache). It can change the `go` and
`toolchain` lines. After the command, examine `git diff go.mod go.sum`. Tell the user about
each change to a directive.

| Task | Command |
|------|---------|
| Add or pin a dependency | `go get example.com/pkg@v1.2.3` |
| Upgrade one dependency | `go get example.com/pkg@latest` |
| Remove a dependency | delete its imports, then `go mod tidy` (or `go get example.com/pkg@none`) |
| Find why the module needs it | `go mod why -m example.com/pkg` |
| Show the current versions | `go list -m all` |
| Track a dev tool (Go 1.24+) | `go get -tool example.com/cmd/tool@version` |

Ask the user before a broad upgrade (`go get -u ./...`). It can add many unrelated changes.

In a workspace (`go.work` is present), run `go mod tidy` in each module directory.
`go work sync` writes the resolved versions of the workspace back into the member modules.
