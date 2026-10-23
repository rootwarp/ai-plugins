---
description: 'Python architecture rules: SOLID principles, layers, dependency direction, and protocol boundaries. Use when you design modules or packages, decide where logic belongs, or review if a change respects the layers.'
name: python-architecture
---

# Architecture

Use a clean (hexagonal) architecture. Express it with Python `Protocol` classes and constructor
injection.

## Layers

| Layer | Responsibility | Can import |
|-------|----------------|------------|
| Domain | Entities, value objects, business rules | Nothing outside the domain |
| Application | Use cases and the ports that they need | Domain |
| Adapters | HTTP and CLI handlers (inbound), DB and API clients (outbound) | Application, Domain |
| Infrastructure | Configuration loading, wiring, framework setup | All layers |

Dependencies point inward. The domain does not use frameworks, I/O, or Pydantic. Thus, you can
test its rules without infrastructure.

## Package Structure

This structure is the target for a service with real domain logic. A small tool or library
starts with the minimal layout in the `python-project-layout` skill. Add layers when a second
adapter or a non-trivial business rule appears. Do not create empty layers.

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

- Declare a port in the module that uses it. Usually this module is `application/ports.py`.
  Declare the port in the domain only when a domain service itself makes the call. Adapters
  import the port. The port never imports an adapter.
- Keep protocols small (1-3 methods). Make larger protocols from small ones: use `UserReader` and
  `UserWriter`, not one large `UserStore`. An implementation must obey the documented contract,
  not only the signatures.
- To extend behavior, add a new implementation of an existing protocol. Do not edit the callers.
- Inject dependencies through `__init__` parameters. Only the composition root
  (`infrastructure/container.py` or `main`) creates concrete adapters.
- Use dataclasses for domain types. Use Pydantic only at the adapter boundary (request models,
  response models, and settings), and convert to domain types there.

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

Look for circular imports between modules. Usually, a circular import shows that a dependency
points in the wrong direction.
