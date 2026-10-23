---
description: 'Format Go sources with golangci-lint fmt, goimports, or go fmt. Use when the user asks you to format Go code.'
name: go-fmt
allowed-tools: Bash(golangci-lint fmt:*), Bash(go fmt:*), Bash(go list:*), Bash(goimports:*), Bash(command -v:*), Read
---

# /go:go-fmt

Format the Go source files with the first option that applies. If the tool of an option is not
installed, go to the next option. Do not install the tool without permission.

The three options format only package files. Thus they do not change `testdata/` fixtures and
`vendor/`. For this reason, do not run `gofmt -w .` or `goimports -w .` on the full tree.

1. **`.golangci.yml` with `version: "2"`:** Apply the formatters that the project configures.
   These include its `goimports` local prefixes:

   ```bash
   golangci-lint fmt
   ```

2. **`goimports` is installed:** Format the files and group the imports. The imports of the
   module go in a separate group. The command gives each file by name, because `goimports`
   goes into each directory that it gets. The command skips files that build constraints
   exclude for the current OS. For those files, use option 1 or option 3.

   ```bash
   F='{{$d := .Dir}}{{range .GoFiles}}{{$d}}/{{.}} {{end}}'
   F+='{{range .TestGoFiles}}{{$d}}/{{.}} {{end}}{{range .XTestGoFiles}}{{$d}}/{{.}} {{end}}'
   goimports -local "$(go list -m)" -w $(go list -f "$F" ./...)
   ```

3. **Other conditions:** Run plain `gofmt` on the packages of the module:

   ```bash
   go fmt ./...
   ```
