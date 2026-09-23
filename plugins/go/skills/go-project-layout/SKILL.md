---
description: 'Go project layout: package and directory structure, cmd and internal placement. Use when adding a package or setting up a project.'
name: go-project-layout
---

# Project Layout

Follow the official [Organizing a Go module](https://go.dev/doc/modules/layout) guide. The
community [golang-standards/project-layout](https://github.com/golang-standards/project-layout)
repo is not an official standard; borrow its support directories below only when a large service
needs them.

## Core Directories

| Directory | Purpose |
|-----------|---------|
| `cmd/` | Main applications; one subdirectory per executable |
| `internal/` | Private code; cannot be imported by external projects |
| `pkg/` | Optional; public library code safe for external import |

## Service Directories

| Directory | Purpose |
|-----------|---------|
| `api/` | OpenAPI specs, protobuf definitions, JSON schemas |
| `web/` | Static assets, templates, SPAs |

## Support Directories

| Directory | Purpose |
|-----------|---------|
| `configs/` | Configuration templates and defaults |
| `scripts/` | Build, install, analysis scripts |
| `build/` | Packaging (`build/package/`) and CI (`build/ci/`) |
| `deployments/` | Docker, Kubernetes, Terraform configs |
| `test/` | External test apps and test data |
| `docs/` | Design and user documentation |
| `tools/` | Supporting tools for the project |
| `examples/` | Example usage code |

## Rules

- Never create `/src` directory - not idiomatic for Go
- Keep `cmd/` minimal; import logic from `internal/` (or the root package of a library)
- Use `internal/` by default; a library exposes its package at the module root — use `pkg/` only
  when there are several public packages and a root package is not enough
- Small projects don't need all directories; start minimal and grow as needed
- Each `cmd/<app>/` should have its own `main.go`

## Recommended Starting Structure

For most projects, start with:

```
project/
├── cmd/
│   └── app/
│       └── main.go
├── internal/
│   └── app/
├── go.mod
└── README.md
```

A library puts its package at the module root instead:

```
project/
├── project.go        # package project
├── project_test.go
├── internal/         # only once private helpers exist
├── go.mod
└── README.md
```

Add directories as needed, not preemptively.
