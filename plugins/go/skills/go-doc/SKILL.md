---
description: 'Look up Go package documentation with go doc or pkg.go.dev. Use when you verify stdlib or third-party Go APIs.'
name: go-doc
argument-hint: '<package> [symbol] [version]'
allowed-tools: Bash(go doc:*), Bash(go list -m:*), Bash(go version:*), WebFetch(domain:pkg.go.dev)
---

# /go:go-doc

Look up the documentation of Go standard library packages and third-party packages.

The arguments are:

- `package`, for example `net/http`.
- `symbol` (optional): a function, a type, or a method.
- `version` (optional): a Go version for the stdlib (`go1.23.0`), or a module version for a
  third-party package (`v1.10.0`).

## Instructions

1. **If the user gives no version, try `go doc` first.** In a module, `go doc` works offline.
   It shows the documentation of the versions that the project builds against. For the stdlib,
   this is the local toolchain. For dependencies, this is the version in `go.mod`.

   ```bash
   go doc encoding/json            # package overview
   go doc encoding/json Marshal    # one symbol
   go doc -all net/http            # everything
   ```

   `go doc` fails for a module that is not in `go.mod` or that is not downloaded. In that
   case, use pkg.go.dev.

2. **pkg.go.dev.** The URL is `https://pkg.go.dev/<package>[@<version>]`. For a third-party
   package, use the full module path. If the user gives no version, select one:
   - Third-party: the version that the project requires,
     `go list -m -f '{{.Version}}' <module>`
   - Stdlib: the local toolchain from `go version`
   - Outside a module, or if the user wants the newest API: do not add `@<version>`

   Use WebFetch with a prompt for the request. An example is "Extract the signature,
   description, parameters, return values, and examples for `<symbol>`". For a full package,
   ask for its overview, its key functions and types, and its examples.

3. **Answer** with the applicable signatures, their documented behavior, and an example if one
   exists. If WebFetch does not find the path, do these steps:
   - Look for typing errors.
   - For the stdlib, try again without the version suffix.
   - Propose the most probable package. A third-party package needs its full module path.
