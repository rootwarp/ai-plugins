---
description: 'Format Go sources with golangci-lint fmt, goimports, or go fmt. Use when asked to format Go code.'
name: go-fmt
allowed-tools: Bash(golangci-lint fmt:*), Bash(go fmt:*), Bash(go list:*), Bash(goimports:*), Bash(command -v:*), Read
---

# /go:go-fmt

Format Go source files.

## Instructions

Pick the first option that applies. All three format package directories only, so `testdata/`
fixtures and `vendor/` are left untouched — never run `gofmt -w .` or `goimports -w .` on the
whole tree.

1. **`.golangci.yml` with `version: "2"` exists** — apply the project's configured formatters
   (including `goimports` local-prefixes):

   ```bash
   golangci-lint fmt
   ```

2. **`goimports` is installed** — format and group imports, keeping the module's own imports in
   a separate group:

   ```bash
   F='{{$d := .Dir}}{{range .GoFiles}}{{$d}}/{{.}} {{end}}'
   F+='{{range .TestGoFiles}}{{$d}}/{{.}} {{end}}{{range .XTestGoFiles}}{{$d}}/{{.}} {{end}}'
   goimports -local "$(go list -m)" -w $(go list -f "$F" ./...)
   ```

   (Files are listed individually because `goimports` recurses into any directory it is given.
   Files excluded by build constraints for the current OS are skipped; use option 1 or 3 for
   those.)

3. **Otherwise** — plain `gofmt` over the module's packages:

   ```bash
   go fmt ./...
   ```

If a tool is missing, use the next option; do not install it without asking the user.
