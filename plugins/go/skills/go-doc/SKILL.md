---
description: 'Look up Go package documentation with go doc or pkg.go.dev. Use when verifying stdlib or third-party Go APIs.'
name: go-doc
argument-hint: '<package> [symbol] [version]'
allowed-tools: Bash(go doc:*), Bash(go list -m:*), Bash(go version:*), WebFetch(domain:pkg.go.dev)
---

# /go:go-doc

Look up Go standard library and third-party package documentation.

Arguments: `package` (e.g. `net/http`), optional `symbol` (function, type, or method), and
optional `version`: a Go version for stdlib (`go1.23.0`) or a module version for third-party
packages (`v1.10.0`).

## Instructions

1. **No explicit version: try `go doc` first.** Inside a module it works offline and documents
   exactly the versions the project builds against (the local toolchain for stdlib, `go.mod` for
   dependencies):

   ```bash
   go doc encoding/json            # package overview
   go doc encoding/json Marshal    # one symbol
   go doc -all net/http            # everything
   ```

   It fails for a module that isn't in `go.mod` or hasn't been downloaded. Fall back to
   pkg.go.dev then.

2. **pkg.go.dev.** The URL is `https://pkg.go.dev/<package>[@<version>]`, using the full module
   path for third-party packages. When no version was given, pick one:
   - Third-party: the version the project requires, `go list -m -f '{{.Version}}' <module>`
   - Stdlib: the local toolchain from `go version`
   - Outside a module, or when the user wants the newest API: omit `@<version>`

   Fetch with WebFetch and a prompt aimed at the request, for example "Extract the signature,
   description, parameters, return values, and examples for `<symbol>`", or for a whole
   package, its overview, key functions and types, and examples.

3. **Answer** with the relevant signatures, their documented behavior, and an example when one
   exists. If the path isn't found, check for typos, retry stdlib without the version suffix, and
   suggest the likely package (a third-party package needs its full module path).
