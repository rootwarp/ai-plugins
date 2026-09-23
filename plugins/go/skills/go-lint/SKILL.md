---
description: 'Run golangci-lint (or go vet as fallback) and fix the reported issues. Use when asked to lint Go code.'
name: go-lint
allowed-tools: Bash(golangci-lint run:*), Bash(golangci-lint version:*), Bash(go vet:*), Bash(command -v:*), Read, Edit
---

# /go:go-lint

Run linters on the Go project and fix what they report.

## Instructions

1. **Check the tool.** If `golangci-lint` is missing, tell the user and run `go vet ./...`
   instead; do not install golangci-lint without asking.

2. **Check the version against the config.** Run `golangci-lint version`. If it is v1 and
   `.golangci.yml` declares `version: "2"` (or v2 with a v1 config), stop and tell the user —
   upgrade golangci-lint, or run `golangci-lint migrate` to convert a v1 config. Without a
   config file, golangci-lint uses its `standard` linter set.

3. **Run the linter:**

   ```bash
   if command -v golangci-lint &> /dev/null; then
       golangci-lint run ./...
   else
       go vet ./...
   fi
   ```

4. **Fix the findings** in the source, one linter rule at a time, then re-run until clean.
   `golangci-lint run --fix ./...` applies the auto-fixable ones. Do not silence a finding with
   `//nolint` unless the user agrees; when you do, name the linter and give a reason
   (`//nolint:gosec // input is a constant`).
