---
description: 'Python security: input validation, secret handling, deserialization and subprocess risks, and dependency audits. Use when you handle untrusted input or credentials, or add dependencies.'
name: python-security
---

# Security

## Input and Output

- Validate untrusted input at the boundary with Pydantic models: type, length, range, and
  pattern. Use allowlists. Do not use a blocklist of attack strings such as `<script`. Escape
  output (template autoescaping) to prevent XSS. Input filters do not prevent it. `EmailStr`
  needs the `pydantic[email]` extra.
- Use parameterized queries (`cursor.execute("... WHERE id = %s", (user_id,))`, SQLAlchemy
  expressions, asyncpg `$1` placeholders). Never make SQL with f-strings or `%`.
- Set a limit on the data that you read from untrusted sources (request bodies, uploads,
  archives).

## Dangerous APIs on Untrusted Data

| Do not use | Use |
|------------|-----|
| `pickle`, `shelve`, `marshal` | JSON, or a Pydantic model |
| `yaml.load` | `yaml.safe_load` |
| `eval`, `exec` | `ast.literal_eval`, or a real parser |
| `subprocess` with `shell=True` | An argument list: `subprocess.run(["ls", "--", name])` |
| `xml.etree` / `xml.dom` on untrusted XML | `defusedxml` |
| `tarfile.extractall()` with no filter | `extractall(filter="data")` (default since 3.14) |
| `random` for tokens or keys | `secrets.token_urlsafe(32)` or `secrets.token_hex(32)` |

For a path from a user, resolve the path first. Then make sure that the path is in the base
directory. `pathlib` alone does not stop `../` escapes or symlink escapes.

```python
def safe_path(base: Path, user_path: str) -> Path:
    root = base.resolve()
    candidate = (root / user_path).resolve()
    if not candidate.is_relative_to(root):
        raise ValueError(f"path escapes {root}: {user_path}")
    return candidate
```

## Secrets

- Do not put secrets in code. Load them from environment variables or from a secret manager
  with `pydantic-settings`. Give them the type `SecretStr`, so that they do not appear in
  `repr()` or in logs. Call `.get_secret_value()` only where the code uses the secret.
- Read `.env` only in local development:

  ```python
  class Settings(BaseSettings):
      model_config = SettingsConfigDict(
          env_file=".env" if os.getenv("APP_ENV") == "dev" else None,
      )
      database_url: SecretStr
  ```

- With strict mypy, add `plugins = ["pydantic.mypy"]` to `[tool.mypy]`. Without it, mypy reports
  that `Settings()` does not have its required fields.
- Hash passwords with `argon2-cffi` (`PasswordHasher().hash` / `.verify`). `verify` raises
  `VerifyMismatchError`. It does not return `False`. Do not use `passlib`, because nobody
  maintains it and it does not work with current `bcrypt` releases.
- Compare secrets with `hmac.compare_digest`. Log security events. Never log secret values.

## Network and Files

- Set timeouts on each HTTP client (`httpx.Timeout(10.0, connect=5.0)`). Set a limit on
  redirects.
- Keep TLS verification on. Never give `verify=False`, except in a local test.
- Create files with least-privilege permissions.

## Scans

The Ruff `S` rules find common problems. The `/python:init-dir` configuration turns them on.
Run the scanners one time with `uvx`. Ask the user before you add them as dev dependencies.

```bash
uvx bandit -r src/

# Audit the locked dependencies; the export includes hashes, so pip-audit can skip resolving
uv export --format requirements-txt --no-emit-project -o requirements-audit.txt
uvx pip-audit -r requirements-audit.txt --disable-pip
rm requirements-audit.txt
```
