---
description: 'Go architecture rules: SOLID principles, layering, dependency direction, interface boundaries. Use when you design packages, decide where logic belongs, or review if a change respects the layers.'
name: go-architecture
---

# Architecture

This project uses clean or hexagonal architecture, adapted to idiomatic Go.

## Layers

| Layer | Responsibility | Can import |
|-------|----------------|------------|
| Domain | Entities and business rules | nothing in the project |
| Application | Use cases, orchestration, outbound ports | Domain |
| Adapters | Inbound handlers (HTTP, gRPC, CLI) and outbound clients | Application, Domain |
| Infrastructure | Config, drivers, framework setup | any layer |

Dependencies point inward. The domain does not import a framework, a database, or a transport.
Thus you can test its logic without infrastructure.

## Target Structure

This structure is for a service with real domain logic. A small tool or library starts with the
minimal layout in the `go-project-layout` skill. Add layers when a second adapter or a
non-trivial business rule appears. Do not make empty layers.

```text
internal/
├── domain/           # entities and rules
├── application/      # use cases; ports.go holds the outbound interfaces they consume
├── adapters/
│   ├── http/         # inbound
│   ├── grpc/         # inbound
│   └── postgres/     # outbound
└── infrastructure/
    └── config/
```

Connect all the parts manually in `cmd/<app>/main.go`:

1. Make the outbound adapters.
2. Give them to the application constructors.
3. Give the application services to the inbound adapters.

You do not need a DI framework or a code generator.

## Interfaces and Dependencies

- Accept interfaces and return concrete types. Inject dependencies through constructors
  (`NewOrderService(repo OrderRepository)`). Then a caller cannot forget a required dependency.
- Declare an interface in the package that uses it. This is usually `application`. Use
  `domain` only when a domain service makes the call itself. Adapters import the port. The
  port never imports an adapter.
- Keep interfaces small (1-3 methods) and compose them. Use `UserReader` and `UserWriter`, not
  one large `UserStore`.
- An implementation must obey the documented contract, not only the signature. For example, an
  `io.Reader` returns `io.EOF` at the end, and `n <= len(p)`.

```go
// application/ports.go: the port is declared where it is consumed
type UserRepository interface {
    Save(ctx context.Context, user *domain.User) error
    FindByID(ctx context.Context, id string) (*domain.User, error)
}

// adapters/postgres/user_repository.go: the adapter implements it, checked at compile time
var _ application.UserRepository = (*UserRepository)(nil)
```

Look for these problems:

- An anemic domain (entities that have no behavior)
- Infrastructure types in the domain
- God packages
- Import cycles
