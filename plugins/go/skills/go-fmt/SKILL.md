---
description: 'Format Go sources with gofmt/goimports. Use when asked to format Go code.'
name: go-fmt
---

# /go-fmt

Format Go source files.

## Instructions

Format all Go files in the project. Prefer `goimports` (it formats like `gofmt` and also fixes
and groups imports); fall back to `gofmt`:

```bash
if command -v goimports &> /dev/null; then
    goimports -w .
else
    gofmt -w .
fi
```

In a project with a golangci-lint v2 config, `golangci-lint fmt` applies the formatters the
project configured (including `goimports` local-prefixes) and is the better choice.
