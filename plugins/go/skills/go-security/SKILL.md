---
description: 'Go security: input validation, secret handling, dependency auditing. Use when handling untrusted input or credentials, or adding dependencies.'
name: go-security
---

# Security

- Never hardcode secrets, credentials, or API keys
- Use environment variables or secure vaults for configuration
- Validate and sanitize ALL user input
- Use parameterized queries to prevent SQL injection
- Escape output to prevent XSS in web applications
- Use `crypto/rand` for cryptographic randomness, not `math/rand`
- Set appropriate timeouts on HTTP clients and servers
- Use TLS for all network communication
- Avoid `unsafe` package unless absolutely necessary
- Run `golangci-lint run` with `govet`, `staticcheck`, and `gosec` enabled (see the `go-lint`
  skill); `go vet ./...` is the fallback
- Keep dependencies updated; audit known vulnerabilities with `govulncheck ./...`
  (`go install golang.org/x/vuln/cmd/govulncheck@latest`). `go mod verify` only checks that
  downloaded modules match `go.sum` — it is an integrity check, not a vulnerability audit
- Bound what you read from untrusted sources: `http.MaxBytesReader`, `io.LimitReader`
- Build file paths from user input with `filepath.Clean` plus a root check, or `os.Root` (Go 1.24+)
- Compare secrets with `crypto/subtle.ConstantTimeCompare`; hash passwords with
  `golang.org/x/crypto/argon2` or `bcrypt`
- Follow principle of least privilege for file permissions
- Log security events but never log sensitive data
