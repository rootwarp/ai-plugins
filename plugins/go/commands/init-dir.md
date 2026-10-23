---
description: Initialize a Go project with standard directories, a Makefile, a .gitignore file, and a golangci-lint v2 config
argument-hint: '[minimal|library|service] [app-name] [module-path]'
allowed-tools: Bash(go mod init:*), Bash(go build:*), Bash(go vet:*), Bash(mkdir:*), Bash(touch:*), Bash(ls:*), Bash(tree:*), Bash(golangci-lint config verify:*), Bash(git remote get-url:*), Read, Write
---

# /go:init-dir

Initialize a Go project with directories, a Makefile, a `.gitignore` file, and a golangci-lint v2
config.

## Step 0: Check Existing State

Before you make files, look for `go.mod`, `Makefile`, `.gitignore`, `.golangci.yml`, `cmd/`,
`internal/`, and `pkg/`.

- If `go.mod` exists, skip Step 3. Use its module path again (`module <path>` on the first line).
- For each other file that exists, ask the user to select one action: skip, merge, or
  overwrite. Do not overwrite a file without an answer.

## Step 1: Gather Information

Ask the user these questions. Skip the questions that `$ARGUMENTS` or Step 0 answered.

1. **Project type:**
   - Minimal (recommended): `cmd/` and `internal/`
   - Library: the package at the module root, and `internal/` only if necessary
   - Full Service: `cmd/` and the layers of `internal/` from the `go-architecture` skill

2. **Application name**: The name of `cmd/<app-name>/` and of the binary. A library uses the
   package name.

3. **Module path**: For example, `github.com/username/project`. If
   `git remote get-url origin` gives a URL, propose a path from it.

## Step 2: Create Directories

The `go-project-layout` skill gives this rule: make only the directories that hold code now.
Make the other directories (`api/`, `configs/`, `deployments/`, and others) when their first
file arrives.

**Minimal Project:**

```bash
mkdir -p cmd/<app-name> internal/<app-name>
```

**Library Project:** Make no directories. The package is at the module root (Step 4).

**Full Service Project:**

```bash
mkdir -p cmd/<app-name> internal/domain internal/application internal/adapters
```

## Step 3: Initialize Go Module

Skip this step if `go.mod` exists.

```bash
go mod init <module-path>
```

If the command fails (for example, an invalid path or a nested module), show the error. Then
ask the user what to do. Do not continue before the user answers.

## Step 4: Create Source Files

For a project that is not a library, make `cmd/<app-name>/main.go`:

```go
package main

func main() {
}
```

For a library project, make `<package-name>.go` at the module root:

```go
// Package <package-name> ...
package <package-name>
```

Then add `.gitkeep` to each directory from Step 2 that is empty. Use a loop on only those
directories. A `find` on the full tree also writes into `.git/` and into unrelated folders.

```bash
for d in <created-dirs>; do
    [ -z "$(ls -A "$d")" ] && touch "$d/.gitkeep"
done
```

## Step 5: Write Makefile

Replace `<app-name>`. The `fmt` target uses `golangci-lint fmt`. Thus it uses the formatters that
Step 7 configures.

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

For a library project, use this Makefile:

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

## Step 6: Write .gitignore

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

## Step 7: Write .golangci.yml

Use the golangci-lint v2 format. Replace `<module-path>`:

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

Some decisions in this config are not obvious:

- `govet` disables `fieldalignment`. This check gives many findings, and it makes code less
  readable to save bytes.
- `govet` disables `shadow`. This check reports the usual reuse of `err`.
- The upstream defaults of staticcheck exclude these `ST` checks. They also overlap with `revive`.
- In v2, `staticcheck` includes `gosimple` and `stylecheck`.
- In v2, `gofmt` and `goimports` are under `formatters`, not under `linters`.

## Step 8: Verify and Show the Result

```bash
go build ./...
```

If `golangci-lint` is installed, also run `golangci-lint config verify`. If it is not installed,
tell the user. Do not install it without permission.

Show the new structure with `tree` (or `ls -R`).
