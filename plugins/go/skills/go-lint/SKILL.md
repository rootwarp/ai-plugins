---
description: 'Run golangci-lint (or go vet as the fallback) and fix the reported issues. Use when the user asks you to lint Go code.'
name: go-lint
allowed-tools: Bash(golangci-lint run:*), Bash(golangci-lint version:*), Bash(go vet:*), Bash(command -v:*), Read, Edit
---

# /go:go-lint

Run the linters on the Go project and fix the problems that they report.

1. **Check the tool.** If `golangci-lint` is not installed (`command -v golangci-lint`), tell
   the user and run `go vet ./...` instead. Do not install golangci-lint without permission.
2. **Compare the versions.** Compare the output of `golangci-lint version` with
   `.golangci.yml`:
   - If a v1 binary finds a `version: "2"` config, stop.
   - If a v2 binary finds a v1 config, stop.
   - Then tell the user to upgrade, or to run `golangci-lint migrate` to convert a v1 config.

   If there is no config file, golangci-lint uses its `standard` linter set.
3. **Run** `golangci-lint run ./...`.
4. **Fix** the findings in the source, one rule at a time. Run the linter again until it
   reports no findings. `golangci-lint run --fix ./...` fixes the findings that have an
   automatic fix. Suppress a finding with `//nolint` only if the user agrees. Then name the
   linter and give a reason (`//nolint:gosec // input is a constant`).
