---
description: 'Run golangci-lint (or go vet as fallback) and fix the reported issues. Use when asked to lint Go code.'
name: go-lint
allowed-tools: Bash(golangci-lint run:*), Bash(golangci-lint version:*), Bash(go vet:*), Bash(command -v:*), Read, Edit
---

# /go:go-lint

Run linters on the Go project and fix what they report.

1. **Check the tool.** If `golangci-lint` is missing (`command -v golangci-lint`), tell the user
   and run `go vet ./...` instead. Don't install golangci-lint without asking.
2. **Match versions.** `golangci-lint version` against `.golangci.yml`: if a v1 binary meets a
   `version: "2"` config, or a v2 binary meets a v1 config, stop and tell the user to upgrade, or
   to run `golangci-lint migrate` to convert a v1 config. Without a config file, golangci-lint
   uses its `standard` linter set.
3. **Run** `golangci-lint run ./...`.
4. **Fix** findings in the source, one rule at a time, and re-run until clean.
   `golangci-lint run --fix ./...` applies the auto-fixable ones. Suppress a finding with
   `//nolint` only when the user agrees, and then name the linter and give a reason
   (`//nolint:gosec // input is a constant`).
