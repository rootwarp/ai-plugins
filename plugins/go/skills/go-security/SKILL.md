---
description: 'Go security: input validation, secret handling, dependency auditing. Use when you handle untrusted input or credentials, or when you add dependencies.'
name: go-security
---

# Security

## Untrusted Input

- Validate input at the boundary. Use only parameterized queries. Do not build SQL with string
  concatenation.
- Render HTML with `html/template`. `text/template` does no escaping.
- Run programs with `exec.Command(name, args...)`. Do not use `sh -c` with input in the string.
- Set a limit on reads: `http.MaxBytesReader`, `io.LimitReader`. Set timeouts on HTTP clients
  and servers. The zero-value `http.Client` and `http.Server` have no timeouts.
- Open files that a user names through `os.Root` (Go 1.24+). `os.Root` also stops symlink
  escapes. If you cannot use it, reject names that fail `filepath.IsLocal`. `filepath.Clean`
  with a prefix check does not stop symlinks.
- Use `unsafe` only with a written reason.

## Secrets and Crypto

- Do not put secrets in code. Read them from the environment or from a secret store. Do not
  log them.
- Use `crypto/rand` for all security-related values. Do not use `math/rand`.
- Compare secrets with `crypto/subtle.ConstantTimeCompare`. Hash passwords with
  `golang.org/x/crypto/argon2` or `bcrypt`.

## Tools

- Lint with `gosec`, `govet`, and `staticcheck` enabled (see the `go-lint` skill). The fallback
  is `go vet ./...`.
- Audit dependencies with `govulncheck ./...`. If it is not installed, ask the user before you
  install it. There are two alternatives:
  - Run it one time: `go run golang.org/x/vuln/cmd/govulncheck@latest ./...`.
  - Add it as a module tool (Go 1.24+): `go get -tool golang.org/x/vuln/cmd/govulncheck`, then
    `go tool govulncheck ./...`.
- `go mod verify` only compares the downloaded modules with `go.sum`. It is an integrity check.
  It is not a vulnerability audit.
