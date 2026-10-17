---
description: 'Go project layout: package and directory structure, cmd and internal placement. Use when adding a package or setting up a project.'
name: go-project-layout
---

# Project Layout

Follow the official [Organizing a Go module](https://go.dev/doc/modules/layout) guide.
[golang-standards/project-layout](https://github.com/golang-standards/project-layout) is not an
official standard. Borrow its support directories (`api/`, `web/`, `configs/`, `scripts/`,
`build/`, `deployments/`, `test/`, `docs/`, `tools/`, `examples/`) only when a large service
needs them.

## Rules

- `cmd/<app>/main.go` per executable, kept thin: it parses config, wires dependencies, and
  calls into `internal/`.
- `internal/` by default for private code.
- A library exposes its package at the module root. Use `pkg/` only when there are several public
  packages and a root package isn't enough.
- No `src/` directory; it isn't idiomatic Go.
- Start minimal and add a directory when its first file arrives, not preemptively.

## Starting Structures

Application:

```text
project/
├── cmd/app/main.go
├── internal/app/
├── go.mod
└── README.md
```

Library:

```text
project/
├── project.go        # package project
├── project_test.go
├── internal/         # only once private helpers exist
├── go.mod
└── README.md
```

A service with real domain logic grows `internal/` into the layers in the `go-architecture`
skill.
