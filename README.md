# ai-plugins

A collection of Claude Code plugins for Go, Rust, Python development and multi-agent team
orchestration.

## Installation

### From GitHub

```bash
# Add the marketplace once
/plugin marketplace add rootwarp/ai-plugins

# Install specific plugin
/plugin install go@rootwarp-ai-plugins
/plugin install rust@rootwarp-ai-plugins
/plugin install python@rootwarp-ai-plugins
/plugin install dev-team@rootwarp-ai-plugins
```

### Local Development

```bash
claude --plugin-dir ./plugins/go
claude --plugin-dir ./plugins/rust
claude --plugin-dir ./plugins/python
claude --plugin-dir ./plugins/dev-team
```

## Available Plugins

### Go Plugin

| Command/Skill | Description |
|---------------|-------------|
| `/go:init-dir` | Initialize Go project with directories, Makefile, and .gitignore |
| `/go-build` | Build the Go project |
| `/go-test` | Run tests with coverage |
| `/go-lint` | Run linters (golangci-lint or go vet) |
| `/go-mod` | Tidy Go modules |
| `/go-fmt` | Format Go source files |
| `/go-doc` | Look up Go documentation |

<details> <summary>Usage Examples</summary>

```bash
# Initialize a new Go project (interactive)
/go:init-dir

# Build the project
/go-build

# Run tests with the race detector and coverage
/go-test

# Run linters (uses golangci-lint if available, otherwise go vet)
/go-lint

# Format code with goimports
/go-fmt

# Tidy module dependencies
/go-mod

# Look up standard library documentation
/go-doc fmt
/go-doc net/http
/go-doc encoding/json Marshal
```

</details>

### Rust Plugin

| Command/Skill | Description |
|---------------|-------------|
| `/rust:init-dir` | Initialize Rust project with Cargo configuration |
| `/cargo-build` | Build the Rust project |
| `/cargo-test` | Run tests |
| `/cargo-clippy` | Run Clippy linter |
| `/cargo-fmt` | Format Rust source files |
| `/cargo-check` | Fast type checking without building |
| `/cargo-doc` | Generate and view documentation |

<details> <summary>Usage Examples</summary>

```bash
# Initialize a new Rust project (interactive)
/rust:init-dir

# Build the project (debug mode)
/cargo-build

# Build for release
/cargo-build --release

# Quick syntax and type check (faster than build)
/cargo-check

# Run all tests
/cargo-test

# Run specific test and show its stdout
/cargo-test test_name -- --nocapture

# Run Clippy linter
/cargo-clippy

# Run Clippy as the pre-commit gate (all targets, warnings as errors)
/cargo-clippy --all-targets --all-features -- -D warnings

# Format code
/cargo-fmt

# Look up crate documentation
/cargo-doc tokio
/cargo-doc serde 1.0.0
/cargo-doc axum Router
```

</details>

### Python Plugin

| Command/Skill | Description |
|---------------|-------------|
| `/python:init-dir` | Initialize Python project with uv and modern tooling |
| `/pytest` | Run tests with pytest |
| `/ruff` | Run Ruff linter and formatter |
| `/mypy` | Run type checking |
| `/uv` | Manage dependencies with uv |
| `/python-doc` | Look up Python documentation |

<details> <summary>Usage Examples</summary>

```bash
# Initialize a new Python project (interactive)
/python:init-dir

# Run tests
/pytest

# Run tests with coverage
/pytest --cov=mypackage --cov-report=html

# Run specific test file or function
/pytest tests/test_main.py
/pytest tests/test_main.py::test_function_name

# Run Ruff linter with auto-fix
/ruff check --fix .

# Format code with Ruff
/ruff format .

# Run type checking
/mypy src/

# Run strict type checking
/mypy --strict src/

# Add dependencies
/uv add fastapi httpx
/uv add --dev pytest-cov mypy

# Remove a dependency
/uv remove httpx

# Sync dependencies from lock file
/uv sync

# Look up documentation
/python-doc asyncio
/python-doc collections Counter
/python-doc asyncio 3.11
```

</details>

### Dev-Team Plugin

Multi-agent development team orchestration with planning and execution pipelines.

| Skill | Description |
|-------|-------------|
| `/dev-plan` | Full planning pipeline: PRD, research, architecture, project plan, issue estimation, and optional GitHub issue creation |
| `/dev-run` | Execution pipeline: implement issues via code-writer with code-reviewer quality gates |

#### Agents

| Agent | Role | Model |
|-------|------|-------|
| `prd-writer` | Gather requirements and write Product Requirements Documents | Opus 5.5 |
| `researcher` | Investigate technologies, landscape, feasibility, and best practices | Opus 5.5 |
| `software-architect` | Design modular system architecture with clear boundaries and interfaces | Opus 5.5 |
| `project-planner` | Plan phases, milestones, and dependencies | Opus 5.5 |
| `issue-estimator` | Turn plan tasks into the fewest sprint-ready issues that cover them | Opus 5.5 |
| `code-writer` | Implement issues via TDD in isolated git worktrees | Opus 5.5 |
| `code-reviewer` | Review lead: code quality + orchestrates security and bug review + merges | Opus 5.5 |
| `security-auditor` | Scan for vulnerabilities and OWASP issues | Opus 5.5 |
| `bug-hunter` | Find potential bugs, edge cases, and race conditions | Opus 5.5 |

All agents run on Claude Opus 5.5 with a 1M context window.

<details> <summary>Usage Examples</summary>

```bash
# Plan a new project from an idea
/dev-plan Build a REST API for task management with user auth

# Plan and file the result as GitHub issues (still asks before creating anything)
/dev-plan Build a REST API for task management with user auth --github

# Implement all issues from a phase
/dev-run plan/issues/01-phase-1-foundation.md

# Implement all planned issues
/dev-run all
```

#### `/dev-plan` Pipeline

1. **Kickoff** - Clarify requirements and size the work: `feature` (a change to an existing
   codebase) or `project`. The scope sets how wide every later stage fans out
2. **PRD** - The lead interviews you; `prd-writer` drafts the Product Requirements Document
3. **Research** - `researcher` investigates technologies and feasibility (1-2 angles for a
   feature, 2-5 for a project)
4. **Architecture** - `software-architect` designs the system (one delta design for a feature;
   three candidates and a judge for a project)
5. **Project Plan** - `project-planner` defines phases with issue-sized tasks (2-3 days each)
6. **Issue Estimation** - `issue-estimator` writes one issue per plan task by default; an
   adversarial check merges undersized issues and splits oversized ones. Issue count is never
   capped — only issue size is constrained
7. **GitHub Issues (optional)** - files one phase issue per phase, with each planned issue created
   as a sub-issue of it
8. **Wrap-up** - Summarizes the artifacts and offers to commit `plan/` to `develop` so `/dev-run`
   worktrees can read it

Stages 2-6 each end in an `LGTM` gate. Subagents cannot talk to you, so every question comes
from the lead. Stage 7 is opt-in: it is skipped unless you
ask for it, and it shows exactly what it will create before creating anything. It requires the
`gh` CLI, authenticated against the target repository.

#### `/dev-run` Pipeline

1. **Load Issues** - Parse phase files and build execution queues
2. **Write** - `code-writer` implements each issue in an isolated worktree
3. **Pre-Review Gate** - Coverage >= 70% on changed files, tests passing, lint clean, all work
   committed, Conventional Commits messages with no plan IDs
4. **Review** - `code-reviewer` runs quality, security, and bug review
5. **Fix Cycle** - Resolve findings; after 3 cycles with Must Fix still open, you choose to keep
   fixing, abort, or merge anyway
6. **Merge** - The lead asks you to approve the merge and the push to `origin/develop`;
   `code-reviewer` then rebases, fast-forwards, and pushes only what you approved

</details>

## Rules

Each language plugin ships its conventions as skills named `<language>-<topic>`, where `<language>`
is `go`, `rust`, or `python` — for example `rust-tdd`. They load when the work matches their
description, not on every session.

| Skill | Covers |
|-------|--------|
| `<language>-code-style` | Language conventions, formatting, naming |
| `<language>-concurrency` | Concurrent programming patterns and safety |
| `<language>-project-layout` | Standard directory structure |
| `<language>-tdd` | Test-Driven Development workflow |
| `<language>-security` | Input validation, secrets management |
| `<language>-architecture` | SOLID principles, Clean Architecture |

## Plugin Structure

```text
plugins/
├── dev-team/
│   ├── .claude-plugin/plugin.json
│   ├── agents/
│   │   ├── bug-hunter.md
│   │   ├── code-reviewer.md
│   │   ├── code-writer.md
│   │   ├── issue-estimator.md
│   │   ├── prd-writer.md
│   │   ├── project-planner.md
│   │   ├── researcher.md
│   │   ├── security-auditor.md
│   │   └── software-architect.md
│   └── skills/
│       ├── dev-plan/
│       └── dev-run/
├── go/
│   ├── .claude-plugin/plugin.json
│   ├── commands/init-dir.md
│   └── skills/
│       ├── go-build/ go-doc/ go-fmt/ go-lint/ go-mod/ go-test/
│       └── go-architecture/ go-code-style/ go-concurrency/
│           go-project-layout/ go-security/ go-tdd/
├── rust/
│   ├── .claude-plugin/plugin.json
│   ├── commands/init-dir.md
│   └── skills/
│       ├── cargo-build/ cargo-check/ cargo-clippy/ cargo-doc/ cargo-fmt/ cargo-test/
│       └── rust-architecture/ rust-code-style/ rust-concurrency/
│           rust-project-layout/ rust-security/ rust-tdd/
└── python/
    ├── .claude-plugin/plugin.json
    ├── commands/init-dir.md
    └── skills/
        ├── mypy/ pytest/ python-doc/ ruff/ uv/
        └── python-architecture/ python-code-style/ python-concurrency/
            python-project-layout/ python-security/ python-tdd/
```

## License

This project is licensed under the [MIT License](LICENSE).
