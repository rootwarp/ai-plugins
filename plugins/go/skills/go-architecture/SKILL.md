---
description: 'Go architecture rules: SOLID principles, layering, dependency direction, interface boundaries. Use when designing packages, choosing where logic belongs, or reviewing whether a change respects the layers.'
name: go-architecture
---

# Architecture

Clean / hexagonal architecture, adapted to idiomatic Go.

## Layers

| Layer | Responsibility | May import |
|-------|----------------|------------|
| Domain | Entities and business rules | nothing in the project |
| Application | Use cases, orchestration, outbound ports | Domain |
| Adapters | Inbound handlers (HTTP, gRPC, CLI); outbound clients | Application, Domain |
| Infrastructure | Config, drivers, framework setup | any layer |

Dependencies point inward. The domain has no framework, database, or transport imports, so its
logic is testable without infrastructure.

## Target Structure

This is the shape for a service with real domain logic. A small tool or library starts with the
minimal layout in the `go-project-layout` skill and grows into layers once a second adapter or a
non-trivial business rule appears. Don't scaffold empty layers.

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

Wire everything by hand in `cmd/<app>/main.go`: build the outbound adapters, pass them to the
application constructors, and hand those to the inbound adapters. A DI framework or code
generator isn't needed.

## Interfaces and Dependencies

- Accept interfaces, return concrete types. Inject dependencies through constructors
  (`NewOrderService(repo OrderRepository)`) so required ones can't be forgotten.
- Declare an interface in the package that consumes it: normally `application`, or `domain`
  only when a domain service makes the call itself. Adapters import the port, never the
  reverse.
- Keep interfaces small (1-3 methods) and compose them (`UserReader` + `UserWriter` instead
  of a fat `UserStore`).
- Implementations honor the documented contract, not just the signature (an `io.Reader`
  returns `io.EOF` at end and `n <= len(p)`).

```go
// application/ports.go: the port is declared where it is consumed
type UserRepository interface {
    Save(ctx context.Context, user *domain.User) error
    FindByID(ctx context.Context, id string) (*domain.User, error)
}

// adapters/postgres/user_repository.go: the adapter implements it, checked at compile time
var _ application.UserRepository = (*UserRepository)(nil)
```

Watch for an anemic domain (entities with no behavior), infrastructure types leaking into the
domain, god packages, and import cycles.
