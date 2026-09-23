---
description: 'Go security: input validation, secret handling, dependency auditing. Use when handling untrusted input or credentials, or adding dependencies.'
name: go-security
---

# Security

- Never hardcode secrets, credentials, or API keys
- Use environment variables or secure vaults for configuration
- Validate and sanitize ALL user input
- Use parameterized queries to prevent SQL injection
- Escape output to prevent XSS: render HTML with `html/template`, never `text/template`
- Run external programs with `exec.Command(name, args...)`, never through `sh -c` with user input
- Use `crypto/rand` for cryptographic randomness, not `math/rand`
- Set appropriate timeouts on HTTP clients and servers
- Use TLS for all network communication
- Avoid `unsafe` package unless absolutely necessary
- Run `golangci-lint run` with `govet`, `staticcheck`, and `gosec` enabled (see the `go-lint`
  skill); `go vet ./...` is the fallback
- Keep dependencies updated; audit known vulnerabilities with `govulncheck ./...`. If it is not
  installed, ask before installing; alternatives are a one-off
  `go run golang.org/x/vuln/cmd/govulncheck@latest ./...` or tracking it as a module tool
  (`go get -tool golang.org/x/vuln/cmd/govulncheck`, then `go tool govulncheck ./...`, Go 1.24+).
  `go mod verify` only checks that downloaded modules match `go.sum` — it is an integrity check,
  not a vulnerability audit
- Bound what you read from untrusted sources: `http.MaxBytesReader`, `io.LimitReader`
- Open files named by user input through `os.Root` (Go 1.24+), which also blocks symlink escapes;
  otherwise reject names that fail `filepath.IsLocal`. `filepath.Clean` plus a prefix check does
  not stop symlinks
- Compare secrets with `crypto/subtle.ConstantTimeCompare`; hash passwords with
  `golang.org/x/crypto/argon2` or `bcrypt`
- Follow principle of least privilege for file permissions
- Log security events but never log sensitive data
