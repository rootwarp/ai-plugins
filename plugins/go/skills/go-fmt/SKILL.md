---
description: 'Format Go sources with golangci-lint fmt, goimports, or go fmt. Use when asked to format Go code.'
name: go-fmt
allowed-tools: Bash(golangci-lint fmt:*), Bash(go fmt:*), Bash(go list:*), Bash(goimports:*), Bash(command -v:*), Read
---

# /go:go-fmt

Format Go source files with the first option that applies. If its tool is missing, fall through
to the next one rather than installing it without asking.

All three format package files only, which leaves `testdata/` fixtures and `vendor/` untouched.
For that reason, don't run `gofmt -w .` or `goimports -w .` on the whole tree.

1. **`.golangci.yml` with `version: "2"`:** apply the project's configured formatters,
   including its `goimports` local-prefixes:

   ```bash
   golangci-lint fmt
   ```

2. **`goimports` installed:** format and group imports, with the module's own imports in a
   separate group. Files are listed individually because `goimports` recurses into any
   directory it's given. Files excluded by build constraints for the current OS are skipped, so
   use option 1 or 3 for those.

   ```bash
   F='{{$d := .Dir}}{{range .GoFiles}}{{$d}}/{{.}} {{end}}'
   F+='{{range .TestGoFiles}}{{$d}}/{{.}} {{end}}{{range .XTestGoFiles}}{{$d}}/{{.}} {{end}}'
   goimports -local "$(go list -m)" -w $(go list -f "$F" ./...)
   ```

3. **Otherwise:** plain `gofmt` over the module's packages:

   ```bash
   go fmt ./...
   ```
