---
description: 'Python architecture rules: SOLID principles, layering, dependency direction, protocol boundaries. Use when designing modules or packages, choosing where logic belongs, or reviewing whether a change respects the layers.'
name: python-architecture
---

# Architecture

Clean / hexagonal architecture, expressed with Python's `Protocol`s and constructor injection.

## Layers

| Layer | Responsibility | May import |
|-------|----------------|------------|
| Domain | Entities, value objects, business rules | Nothing outside the domain |
| Application | Use cases; declares the ports it needs | Domain |
| Adapters | HTTP/CLI handlers (inbound), DB/API clients (outbound) | Application, Domain |
| Infrastructure | Config loading, wiring, framework setup | All layers |

Dependencies point inward. The domain stays free of frameworks, I/O, and Pydantic, so its rules
are testable without infrastructure.

## Package Structure

This is the target for a service with real domain logic. A small tool or library starts with the
minimal layout in the `python-project-layout` skill and grows into layers when a second adapter
or a non-trivial business rule appears; don't scaffold empty layers.

```text
src/mypackage/
├── domain/               # user.py, order.py
├── application/
│   ├── ports.py          # outbound ports: protocols the use cases consume
│   └── user_service.py
├── adapters/
│   ├── api/              # FastAPI/Flask handlers (inbound)
│   ├── cli/              # CLI commands (inbound)
│   └── postgres/         # repository implementations (outbound)
└── infrastructure/
    ├── config.py
    └── container.py      # composition root
```

## Ports and Injection

- Declare a port where it is consumed (normally `application/ports.py`; the domain only when a
  domain service itself makes the call). Adapters import the port, never the reverse.
- Keep protocols small (1-3 methods) and compose larger ones from them (`UserReader` +
  `UserWriter` rather than a fat `UserStore`). Implementations honor the documented contract,
  not just the signatures.
- Extend by adding a new implementation of an existing protocol rather than editing callers.
- Inject dependencies through `__init__` parameters. Only the composition root
  (`infrastructure/container.py` or `main`) constructs concrete adapters.
- Use dataclasses for domain types; keep Pydantic at the adapter boundary (request and response
  models, settings) and map to domain types there.

```python
# application/ports.py
class UserRepository(Protocol):
    async def save(self, user: User) -> None: ...
    async def find_by_id(self, id: str) -> User | None: ...


# adapters/postgres/users.py: satisfies the port structurally; no subclassing needed
@dataclass
class PostgresUserRepository:
    pool: asyncpg.Pool

    async def save(self, user: User) -> None: ...
    async def find_by_id(self, id: str) -> User | None: ...
```

Watch for circular imports between modules; they usually mean a dependency points the wrong way.
