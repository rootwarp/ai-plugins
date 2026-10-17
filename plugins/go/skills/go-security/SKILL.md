---
description: 'Go security: input validation, secret handling, dependency auditing. Use when handling untrusted input or credentials, or adding dependencies.'
name: go-security
---

# Security

## Untrusted Input

- Validate at the boundary. Use parameterized queries only, never SQL built by string
  concatenation.
- Render HTML with `html/template`; `text/template` does no escaping.
- Run programs as `exec.Command(name, args...)`, never `sh -c` with input in the string.
- Bound reads: `http.MaxBytesReader`, `io.LimitReader`. Set timeouts on HTTP clients and servers,
  since the zero-value `http.Client` and `http.Server` have none.
- Open user-named files through `os.Root` (Go 1.24+), which also blocks symlink escapes.
  Otherwise reject names that fail `filepath.IsLocal`. `filepath.Clean` plus a prefix check
  doesn't stop symlinks.
- Use `unsafe` only with a stated reason.

## Secrets and Crypto

- No secrets in code. Read them from the environment or a secret store, and never log them.
- `crypto/rand` for anything security-relevant, never `math/rand`.
- Compare secrets with `crypto/subtle.ConstantTimeCompare`. Hash passwords with
  `golang.org/x/crypto/argon2` or `bcrypt`.

## Tooling

- Lint with `gosec`, `govet`, and `staticcheck` enabled (see the `go-lint` skill). `go vet ./...`
  is the fallback.
- Audit dependencies with `govulncheck ./...`. If it isn't installed, ask before installing it.
  Alternatives are a one-off `go run golang.org/x/vuln/cmd/govulncheck@latest ./...` or a module
  tool (`go get -tool golang.org/x/vuln/cmd/govulncheck`, then `go tool govulncheck ./...`, Go
  1.24+).
- `go mod verify` only checks downloaded modules against `go.sum`. It's an integrity check, not
  a vulnerability audit.
