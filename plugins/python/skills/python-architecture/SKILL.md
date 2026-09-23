---
description: 'Python architecture rules: SOLID principles, layering, dependency direction, protocol boundaries. Use when designing modules or packages, choosing where logic belongs, or reviewing whether a change respects the layers.'
name: python-architecture
---

# Architecture

Follow [Clean
Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) and
[Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/) principles adapted
for Python's dynamic nature and protocol system.

## SOLID Principles

- **Single Responsibility** — each module does one thing well; split classes with multiple reasons
  to change; keep business logic separate from infrastructure (logging, HTTP, database)
- **Open/Closed** — extend by adding new implementations of existing `Protocol`s (e.g. a new
  `Notifier`), not by modifying existing code; prefer composition over inheritance
- **Liskov Substitution** — implementations must honor the protocol's documented contract (e.g. a
  `Reader.read()` returns empty bytes at EOF)
- **Interface Segregation** — keep protocols small (1-3 methods); define them at the consumer, not
  the provider; compose larger protocols from smaller ones (`UserReader` + `UserWriter` instead of
  a fat `UserStore`)
- **Dependency Inversion** — depend on `Protocol`s, not concrete types; inject dependencies through
  `__init__` parameters instead of constructing them internally

## Layered Architecture

Organize code into distinct layers with clear responsibilities:

| Layer | Responsibility | Dependencies |
|-------|---------------|--------------|
| Domain | Business entities and rules | None |
| Application | Use cases, orchestration | Domain |
| Adapters | External system integration | Application, Domain |
| Infrastructure | Frameworks, drivers, tools | All layers |

### Dependency Rule

Dependencies point inward only:
- Domain layer has no external dependencies
- Application layer depends only on Domain
- Adapters depend on Application and Domain
- Infrastructure can depend on all layers

## Recommended Package Structure

This is the target for a service with real domain logic. A small tool or library starts with the
minimal layout in the `python-project-layout` skill and grows into layers when a second adapter
or a non-trivial business rule appears — do not scaffold empty layers.

```
src/mypackage/
├── __init__.py
├── domain/               # Business entities and rules
│   ├── __init__.py
│   ├── user.py           # User entity
│   └── order.py          # Order entity
├── application/          # Use cases / services
│   ├── __init__.py
│   ├── ports.py          # Outbound ports: the protocols the use cases consume
│   ├── user_service.py
│   └── order_service.py
├── adapters/
│   ├── __init__.py
│   ├── api/              # FastAPI/Flask handlers (inbound)
│   ├── cli/              # CLI commands (inbound)
│   └── postgres/         # Database implementation (outbound)
└── infrastructure/
    ├── __init__.py
    ├── config.py         # Configuration loading
    └── container.py      # Dependency injection
```

## Ports and Adapters

### Inbound Ports (Driving)

Entry points that invoke domain logic:
- HTTP/REST handlers (FastAPI, Flask)
- CLI commands (Click, Typer)
- Message queue consumers
- GraphQL resolvers

### Outbound Ports (Driven)

Protocols the use cases need, declared by the module that calls them (normally `application`;
`domain` only when a domain service itself makes the call):
- Repository protocols
- External API clients
- Message publishers
- Cache protocols

```python
from typing import Protocol
from dataclasses import dataclass

# Outbound port (application/ports.py — declared where it is consumed)
class UserRepository(Protocol):
    async def save(self, user: User) -> None: ...
    async def find_by_id(self, id: str) -> User | None: ...

# Adapter implements the port
@dataclass
class PostgresUserRepository:
    pool: asyncpg.Pool

    async def save(self, user: User) -> None:
        # PostgreSQL-specific implementation
        await self.pool.execute(
            "INSERT INTO users (id, name, email) VALUES ($1, $2, $3)",
            user.id, user.name, user.email,
        )

    async def find_by_id(self, id: str) -> User | None:
        row = await self.pool.fetchrow(
            "SELECT * FROM users WHERE id = $1", id
        )
        return User(**row) if row else None
```

## Dependency Injection

```python
# infrastructure/container.py
from dataclasses import dataclass
from mypackage.application.ports import UserRepository
from mypackage.application.user_service import UserService
from mypackage.adapters.postgres import PostgresUserRepository

@dataclass
class Container:
    """Dependency injection container."""
    user_repo: UserRepository
    user_service: UserService

    @classmethod
    async def create(cls, config: Config) -> "Container":
        pool = await asyncpg.create_pool(config.database_url)
        user_repo = PostgresUserRepository(pool)
        user_service = UserService(user_repo)
        return cls(user_repo=user_repo, user_service=user_service)
```

## Best Practices

- Keep the domain layer pure; no framework dependencies
- Use dependency injection for all external dependencies
- Define protocols in the module that uses them; adapters import the port, never the reverse
- Test domain logic without infrastructure
- Use dataclasses for domain value objects; keep Pydantic at the adapter boundary (request and
  response models, settings) and map to domain types there
- Avoid circular imports between modules

## Anti-Patterns to Avoid

- **Anemic Domain Model**: Domain objects with only data, no behavior
- **Big Ball of Mud**: No clear boundaries between concerns
- **God Module**: One module that does everything
- **Leaky Abstractions**: Infrastructure details bleeding into domain
- **Circular Imports**: Modules that import each other

---

## Architecture Checklist

Before finalizing design decisions, verify:

- [ ] Domain layer has no external dependencies
- [ ] Dependencies point inward (toward domain)
- [ ] Protocols defined at consumer, not provider
- [ ] Each module has a single, clear responsibility
- [ ] External systems accessed through protocols
- [ ] Business logic testable without infrastructure
- [ ] No circular imports between modules
- [ ] Dependency injection used for external dependencies
