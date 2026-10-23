---
description: 'Go project layout: package and directory structure, cmd and internal placement. Use when you add a package or set up a project.'
name: go-project-layout
---

# Project Layout

Follow the official [Organizing a Go module](https://go.dev/doc/modules/layout) guide.
[golang-standards/project-layout](https://github.com/golang-standards/project-layout) is not an
official standard. Use its support directories (`api/`, `web/`, `configs/`, `scripts/`,
`build/`, `deployments/`, `test/`, `docs/`, `tools/`, `examples/`) only if a large service
needs them.

## Rules

- Give each executable a `cmd/<app>/main.go`. Keep it small: it reads the config, connects the
  dependencies, and calls code in `internal/`.
- By default, put private code in `internal/`.
- A library puts its package at the module root. Use `pkg/` only if there are many public
  packages and a root package is not sufficient.
- Do not make a `src/` directory. It is not idiomatic Go.
- Start with a minimal structure. Add a directory when its first file arrives, not before.

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

For a service with real domain logic, divide `internal/` into the layers in the
`go-architecture` skill.
