---
description: 'Python security: input validation, secret handling, deserialization and subprocess risks, dependency auditing. Use when handling untrusted input or credentials, or adding dependencies.'
name: python-security
---

# Security

- Never hardcode secrets, credentials, or API keys
- Use environment variables or secret managers for configuration
- Validate and sanitize ALL user input
- Use parameterized queries to prevent SQL injection (SQLAlchemy, asyncpg)
- Escape output properly to prevent XSS in web applications
- Use `secrets` module for cryptographic randomness, not `random`
- Set appropriate timeouts on HTTP clients and network operations
- Use TLS/HTTPS for all network communication
- Run Ruff's `S` rules (on in the `/python:init-dir` config) or `uvx bandit` to catch common
  security issues
- Keep dependencies updated; audit with `uvx pip-audit` (see Security Scanning)
- Follow principle of least privilege for file permissions
- Log security events but never log sensitive data
- Load `.env` files in local development only, not production

## Input Validation

`EmailStr` needs the `email` extra (`pydantic[email]`, which installs `email-validator`).

```python
from pydantic import BaseModel, EmailStr, Field, field_validator

class UserInput(BaseModel):
    """Validated user input."""
    name: str = Field(min_length=1, max_length=100)
    email: EmailStr
    age: int = Field(ge=0, le=150)

    @field_validator("name")
    @classmethod
    def name_is_printable(cls, v: str) -> str:
        v = v.strip()
        if not v or not v.isprintable():
            raise ValueError("name must be non-empty printable text")
        return v
```

Validate shape with an allowlist (type, length, pattern); do not try to blocklist attack strings
such as `<script` — XSS is prevented by escaping on output, not by filtering input.

## SQL Injection Prevention

```python
# Bad: string formatting
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")

# Good: parameterized query
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))

# Good: SQLAlchemy
from sqlalchemy import select
stmt = select(User).where(User.id == user_id)
result = session.execute(stmt)
```

## Secrets Management

```python
import os
from functools import lru_cache

from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    """Application settings from environment variables."""

    model_config = SettingsConfigDict(
        # Read .env only in local development; production uses real env vars
        env_file=".env" if os.getenv("APP_ENV") == "dev" else None,
        env_file_encoding="utf-8",
    )

    # SecretStr keeps values out of repr() and logs
    database_url: SecretStr
    api_key: SecretStr
    secret_key: SecretStr

@lru_cache
def get_settings() -> Settings:
    return Settings()

# Usage: unwrap only where the secret is consumed
api_key = get_settings().api_key.get_secret_value()
```

With strict mypy, add `plugins = ["pydantic.mypy"]` to `[tool.mypy]`; without it mypy reports
`Settings()` as missing its required fields.

## Cryptographic Randomness

```python
import secrets

# Good: cryptographically secure
token = secrets.token_urlsafe(32)
api_key = secrets.token_hex(32)

# Bad: not secure for secrets
import random
token = ''.join(random.choices('abc123', k=32))  # Predictable!
```

## Password Hashing

Use `argon2-cffi` (`passlib` is unmaintained and breaks with current `bcrypt` releases):

```python
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

_hasher = PasswordHasher()

def hash_password(password: str) -> str:
    return _hasher.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    try:
        return _hasher.verify(hashed_password, plain_password)
    except VerifyMismatchError:
        return False
```

## HTTP Client Security

```python
import httpx

# Set timeouts to prevent hanging
client = httpx.Client(
    timeout=httpx.Timeout(10.0, connect=5.0),
    follow_redirects=True,
    max_redirects=5,
)

# Verify SSL certificates (default, but be explicit)
async with httpx.AsyncClient(verify=True) as client:
    response = await client.get("https://api.example.com")
```

## Security Scanning

Run the tools one-off with `uvx`; ask the user before adding them as dev dependencies.

```bash
# Static analysis for security issues
uvx bandit -r src/

# Check the locked dependencies for known vulnerabilities (the export includes hashes, so
# pip-audit can skip resolving them with pip)
uv export --format requirements-txt --no-emit-project -o requirements-audit.txt
uvx pip-audit -r requirements-audit.txt --disable-pip
rm requirements-audit.txt

# In CI/CD
uvx bandit -r src/ -f json -o bandit-report.json
```

## Common Vulnerabilities to Avoid

| Vulnerability | Prevention |
|--------------|------------|
| SQL Injection | Parameterized queries |
| XSS | Escape output, use templating engines |
| CSRF | Use CSRF tokens |
| Path Traversal | Resolve the path and check it stays under the base directory |
| Command Injection | Avoid `shell=True`; pass an argument list |
| Deserialization | Don't unpickle untrusted data |
| Hardcoded Secrets | Environment variables, secret managers |

```python
# Bad: command injection risk
import subprocess
subprocess.run(f"ls {user_input}", shell=True)  # Dangerous!

# Good: no shell, list arguments; "--" stops user input being read as an option
subprocess.run(["ls", "--", user_input], check=True)
```

```python
from pathlib import Path

# Path traversal: resolve, then check containment (pathlib alone does not prevent it)
def safe_path(base: Path, user_path: str) -> Path:
    root = base.resolve()
    candidate = (root / user_path).resolve()
    if not candidate.is_relative_to(root):
        raise ValueError(f"path escapes {root}: {user_path}")
    return candidate
```
