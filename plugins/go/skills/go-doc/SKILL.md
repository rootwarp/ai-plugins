---
description: 'Look up Go package documentation with go doc or pkg.go.dev. Use when verifying stdlib or third-party Go APIs.'
name: go-doc
argument-hint: '<package> [symbol] [version]'
allowed-tools: Bash(go doc:*), Bash(go list -m:*), Bash(go version:*), WebFetch(domain:pkg.go.dev)
---

# /go:go-doc

Look up Go standard library and third-party package documentation, locally with `go doc` or from
pkg.go.dev.

## Arguments

- `package`: The Go package path to look up (e.g., `fmt`, `net/http`, `encoding/json`)
- `version` (optional): Go version for stdlib (e.g., `go1.23.0`) or module version for
  third-party packages (e.g., `v1.10.0`). Defaults to the version the project uses (see below),
  or latest outside a module.
- `symbol` (optional): Specific function, type, or method to focus on

## Instructions

When the user wants to verify or look up Go package documentation:

1. **Try `go doc` first.** Inside a module it is offline and documents exactly the versions the
   project builds against (the local toolchain for stdlib, `go.mod` for dependencies):

   ```bash
   go doc encoding/json            # package overview
   go doc encoding/json Marshal    # one symbol
   go doc -all net/http            # everything
   ```

   It fails for a module that is not in `go.mod` or not yet downloaded — then fall back to
   pkg.go.dev.

2. **Pick the version** for pkg.go.dev when none was given:
   - Third-party: the version the project requires — `go list -m -f '{{.Version}}' <module>`
   - Stdlib: the local toolchain from `go version` (e.g. `go1.26.0`)
   - Outside a module, or when the user asks for the newest API: omit `@{version}` (latest)

3. **Construct the URL** based on the package path:
   - Base URL: `https://pkg.go.dev`
   - Standard library: `https://pkg.go.dev/{package}`
   - With version: `https://pkg.go.dev/{package}@{version}`

4. **Fetch the documentation** using WebFetch with a prompt tailored to the user's needs.

5. **Present the information** including:
   - Package overview and purpose
   - Relevant functions, types, or interfaces
   - Usage examples when available
   - Format verbs or special syntax (if applicable)

## URL Structure Reference

| Package Type | URL Format | Example |
|--------------|------------|---------|
| Top-level | `https://pkg.go.dev/{pkg}` | `https://pkg.go.dev/fmt` |
| Nested | `https://pkg.go.dev/{pkg}/{subpkg}` | `https://pkg.go.dev/net/http` |
| With version | `https://pkg.go.dev/{pkg}@go1.23.0` | `https://pkg.go.dev/fmt@go1.23.0` |
| Third-party | `https://pkg.go.dev/{full-module-path}` | `https://pkg.go.dev/github.com/gin-gonic/gin` |

## Documentation Sections

pkg.go.dev documentation pages typically contain:

- **Overview**: Package purpose and general usage patterns
- **Index**: Quick navigation to functions, types, and examples
- **Examples**: Runnable code samples with expected output
- **Functions**: Function signatures with descriptions
- **Types**: Struct and interface definitions with methods

## Example Usage

### Look up fmt package
```
/go:go-doc fmt
```
Runs `go doc fmt`; falls back to `https://pkg.go.dev/fmt`

### Look up net/http with specific version
```
/go:go-doc net/http go1.22.0
```
Fetches: `https://pkg.go.dev/net/http@go1.22.0` (an explicit version goes straight to pkg.go.dev)

### Look up specific function
```
/go:go-doc encoding/json Marshal
```
Runs `go doc encoding/json Marshal` and focuses on the `Marshal` function.

## Execution

When `go doc` is not enough, use WebFetch to retrieve the documentation (append `@{version}` to
the URL when step 2 chose one):

```
WebFetch url="https://pkg.go.dev/{package}" prompt="Extract the documentation for this Go package. Include: 1) Package overview 2) Key functions and their signatures 3) Important types and interfaces 4) Usage examples. If a specific symbol was requested, focus on that symbol's documentation."
```

If the user asks about a specific symbol (function, type, method), include it in the prompt:

```
WebFetch url="https://pkg.go.dev/{package}" prompt="Find the documentation for the '{symbol}' function/type in this package. Include its signature, description, parameters, return values, and any available examples."
```

## Error Handling

If the package is not found or the URL returns an error:
1. Verify the package path is correct (check for typos)
2. For standard library packages, try without a version suffix
3. Suggest related packages if the exact match isn't found
4. Check if it might be a third-party package requiring full module path
