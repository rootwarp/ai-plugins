---
description: Initialize a Go project with standard directories, Makefile, .gitignore, and golangci-lint v2 config
argument-hint: '[minimal|library|service] [app-name] [module-path]'
allowed-tools: Bash(go mod init:*), Bash(go build:*), Bash(go vet:*), Bash(mkdir:*), Bash(touch:*), Bash(ls:*), Bash(tree:*), Bash(golangci-lint config verify:*), Bash(git remote get-url:*), Read, Write
---

# /go:init-dir

Initialize a standard Go project with directory structure, Makefile, .gitignore, and
golangci-lint v2 config.

## Instructions

### Step 0: Check Existing State

Before creating anything, check the current directory for `go.mod`, `Makefile`, `.gitignore`,
`.golangci.yml`, and `cmd/`, `internal/`, `pkg/`.

- If `go.mod` exists, skip Step 3 and reuse its module path (first line, `module <path>`).
- For every other file that already exists, ask the user per file: skip, merge, or overwrite.
  Never overwrite without an answer.

### Step 1: Gather Information

Ask the user (skip what `$ARGUMENTS` or Step 0 already answered):

1. **Project type:**
   - Minimal (recommended) - `cmd/` and `internal/`
   - Library - package at the module root; `internal/` only when needed
   - Full Service - `cmd/` plus the layered `internal/` from the `go-architecture` skill

2. **Application name** (used for `cmd/<app-name>/` and binary name; libraries use the package
   name instead)

3. **Module path** (e.g., `github.com/username/project`; suggest one from `git remote get-url
   origin` when available)

### Step 2: Create Directories

Follow the `go-project-layout` skill's conventions: create only directories that will hold code
now; add the rest (`api/`, `configs/`, `deployments/`, ...) when their first file arrives.

**Minimal Project:**

```bash
mkdir -p cmd/<app-name> internal/<app-name>
```

**Library Project:** no directories; the package lives at the module root (Step 4).

**Full Service Project:**

```bash
mkdir -p cmd/<app-name> internal/domain internal/application internal/adapters
```

Add `.gitkeep` only to the directories you just created that are still empty — never run a
tree-wide `find`, which would also write into `.git/` and unrelated folders:

```bash
for d in <created-dirs>; do
    [ -z "$(ls -A "$d")" ] && touch "$d/.gitkeep"
done
```

### Step 3: Initialize Go Module

Skip if `go.mod` already exists.

```bash
go mod init <module-path>
```

If it fails (invalid path, nested module), show the error and ask the user how to proceed; do
not continue to later steps.

### Step 4: Create Source Files

For non-library projects, create `cmd/<app-name>/main.go`:

```go
package main

func main() {
}
```

For library projects, create `<package-name>.go` at the module root:

```go
// Package <package-name> ...
package <package-name>
```

### Step 5: Write Makefile

Create `Makefile` with the following content (replace `<app-name>` with actual name). `fmt`
uses `golangci-lint fmt` so formatting follows the formatters configured in Step 7.

```makefile
APP_NAME := <app-name>
BUILD_DIR := bin
GO := go

.PHONY: all build run test test-coverage lint fmt clean mod-tidy help

all: build

build: ## Build the application
	$(GO) build -o $(BUILD_DIR)/$(APP_NAME) ./cmd/$(APP_NAME)

run: build ## Run the application
	./$(BUILD_DIR)/$(APP_NAME)

test: ## Run tests (-race needs cgo and a C compiler)
	$(GO) test -v -race ./...

test-coverage: ## Run tests with coverage
	$(GO) test -v -race -coverprofile=coverage.out ./...
	$(GO) tool cover -html=coverage.out -o coverage.html

lint: ## Run linter
	golangci-lint run ./...

fmt: ## Format code with the formatters in .golangci.yml
	golangci-lint fmt

clean: ## Clean build artifacts
	rm -rf $(BUILD_DIR)
	rm -f coverage.out coverage.html

mod-tidy: ## Tidy go modules
	$(GO) mod tidy

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-15s\033[0m %s\n", $$1, $$2}'
```

For library projects, use this Makefile instead:

```makefile
GO := go

.PHONY: all test test-coverage lint fmt clean mod-tidy help

all: test

test: ## Run tests (-race needs cgo and a C compiler)
	$(GO) test -v -race ./...

test-coverage: ## Run tests with coverage
	$(GO) test -v -race -coverprofile=coverage.out ./...
	$(GO) tool cover -html=coverage.out -o coverage.html

lint: ## Run linter
	golangci-lint run ./...

fmt: ## Format code with the formatters in .golangci.yml
	golangci-lint fmt

clean: ## Clean artifacts
	rm -f coverage.out coverage.html

mod-tidy: ## Tidy go modules
	$(GO) mod tidy

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-15s\033[0m %s\n", $$1, $$2}'
```

### Step 6: Write .gitignore

Create `.gitignore`:

```gitignore
# Binaries
bin/
*.exe
*.exe~
*.dll
*.so
*.dylib

# Test artifacts
*.test
coverage.out
coverage.html

# Dependency directories
vendor/

# IDE
.idea/
.vscode/
*.swp
*.swo
*~

# OS files
.DS_Store
Thumbs.db

# Environment files
.env
.env.local
*.env

# Build artifacts
dist/
build/output/

# Temporary files
tmp/
*.tmp
*.log
```

### Step 7: Write .golangci.yml

Create `.golangci.yml` using the golangci-lint **v2** configuration format (replace
`<module-path>` with the actual module path):

```yaml
version: "2"

run:
  timeout: 5m

linters:
  default: none
  enable:
    - errcheck
    - govet
    - staticcheck
    - ineffassign
    - unused
    - errorlint
    - gosec
    - revive
  settings:
    errcheck:
      check-type-assertions: true
    govet:
      enable-all: true
      disable:
        - fieldalignment
        - shadow
    staticcheck:
      checks: ["all", "-ST1000", "-ST1003", "-ST1016", "-ST1020", "-ST1021", "-ST1022"]

formatters:
  enable:
    - gofmt
    - goimports
  settings:
    goimports:
      local-prefixes:
        - <module-path>

issues:
  max-issues-per-linter: 0
  max-same-issues: 0
```

**Linter rationale:**
- `errcheck`: Catches unchecked errors (aligns with error handling rules)
- `govet`: Detects suspicious constructs; `fieldalignment` (noisy, trades readability for bytes)
  and `shadow` (flags ordinary `err` reuse) are disabled
- `staticcheck`: Comprehensive static analysis (absorbed `gosimple` and `stylecheck` in v2); the
  excluded `ST` checks are upstream's defaults and overlap with `revive`
- `ineffassign/unused`: Identifies dead or unused code
- `errorlint`: Ensures proper error wrapping with `%w`
- `gosec`: Catches security issues
- `revive`: Extensible linter with sensible defaults (golint replacement)
- `gofmt/goimports`: Enforce formatting — configured under `formatters` in v2, not `linters`

### Step 8: Verify and Display Result

```bash
go build ./...
```

If `golangci-lint` is installed, also run `golangci-lint config verify`. If it is missing, tell
the user; do not install it without asking.

Show the created structure using `tree` (or `ls -R` if `tree` is missing) and confirm completion.

## Rules

- Never create `/src` directory
- Never overwrite an existing file without the user's answer from Step 0
- Keep `cmd/` minimal; put logic in `internal/` (or the root package for libraries)
- Use `internal/` for private code
- Use `pkg/` only when external import is intended and the root package is not enough
