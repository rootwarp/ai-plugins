---
description: 'Python security: input validation, secret handling, deserialization and subprocess risks, dependency auditing. Use when handling untrusted input or credentials, or adding dependencies.'
name: python-security
---

# Security

## Input and Output

- Validate untrusted input at the boundary with Pydantic models: type, length, range, pattern.
  Use allowlists. Don't blocklist attack strings such as `<script`, because XSS is prevented by
  escaping on output (template autoescaping), not by filtering input. `EmailStr` needs the
  `pydantic[email]` extra.
- Use parameterized queries (`cursor.execute("... WHERE id = %s", (user_id,))`, SQLAlchemy
  expressions, asyncpg `$1` placeholders); never build SQL with f-strings or `%`.
- Bound what you read from untrusted sources (request bodies, uploads, archives).

## Dangerous APIs on Untrusted Data

| Avoid | Use instead |
|-------|-------------|
| `pickle`, `shelve`, `marshal` | JSON, or a Pydantic model |
| `yaml.load` | `yaml.safe_load` |
| `eval`, `exec` | `ast.literal_eval`, or a real parser |
| `subprocess` with `shell=True` | An argument list: `subprocess.run(["ls", "--", name])` |
| `xml.etree` / `xml.dom` on untrusted XML | `defusedxml` |
| `tarfile.extractall()` with no filter | `extractall(filter="data")` (default since 3.14) |
| `random` for tokens or keys | `secrets.token_urlsafe(32)` or `secrets.token_hex(32)` |

User-supplied paths: resolve, then check containment, because `pathlib` alone does not stop
`../` or symlink escapes:

```python
def safe_path(base: Path, user_path: str) -> Path:
    root = base.resolve()
    candidate = (root / user_path).resolve()
    if not candidate.is_relative_to(root):
        raise ValueError(f"path escapes {root}: {user_path}")
    return candidate
```

## Secrets

- Keep secrets out of code; load them from environment variables or a secret manager with
  `pydantic-settings`, typed as `SecretStr` so they stay out of `repr()` and logs, and unwrap with
  `.get_secret_value()` only where the secret is consumed.
- Read `.env` in local development only:

  ```python
  class Settings(BaseSettings):
      model_config = SettingsConfigDict(
          env_file=".env" if os.getenv("APP_ENV") == "dev" else None,
      )
      database_url: SecretStr
  ```

- Under strict mypy, add `plugins = ["pydantic.mypy"]` to `[tool.mypy]`; without it mypy reports
  `Settings()` as missing its required fields.
- Hash passwords with `argon2-cffi` (`PasswordHasher().hash` / `.verify`; `verify` raises
  `VerifyMismatchError` instead of returning `False`). `passlib` is unmaintained and breaks with
  current `bcrypt` releases.
- Compare secrets with `hmac.compare_digest`. Log security events, never secret values.

## Network and Files

- Set timeouts on every HTTP client (`httpx.Timeout(10.0, connect=5.0)`) and cap redirects.
- Keep TLS verification on; never pass `verify=False` outside a local test.
- Create files with least-privilege permissions.

## Scanning

Ruff's `S` rules (on in the `/python:init-dir` config) catch common issues. Run the scanners
one-off with `uvx`; ask before adding them as dev dependencies.

```bash
uvx bandit -r src/

# Audit the locked dependencies; the export includes hashes, so pip-audit can skip resolving
uv export --format requirements-txt --no-emit-project -o requirements-audit.txt
uvx pip-audit -r requirements-audit.txt --disable-pip
rm requirements-audit.txt
```
