---
description: 'Go architecture rules: SOLID principles, layering, dependency direction, interface boundaries. Use when designing packages, choosing where logic belongs, or reviewing whether a change respects the layers.'
name: go-architecture
---

# Architecture

Follow [Clean
Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) and
[Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/) principles adapted
for Go's idiomatic style.

## SOLID Principles

- **Single Responsibility** — each package does one thing well; split structs with multiple
  reasons to change; keep business logic separate from infrastructure (logging, HTTP, database)
- **Open/Closed** — extend by adding new implementations of existing interfaces (e.g. a new
  `Notifier`), not by modifying existing code; prefer composition over inheritance
- **Liskov Substitution** — implementations must honor the interface's documented contract (e.g.
  an `io.Reader` returns `io.EOF` at end and `n <= len(p)`)
- **Interface Segregation** — keep interfaces small (1-3 methods); define them at the consumer,
  not the provider; compose larger interfaces from smaller ones (`UserReader` + `UserWriter`
  instead of a fat `UserStore`)
- **Dependency Inversion** — accept interfaces, return concrete types; inject dependencies through
  constructors (`NewOrderService(repo OrderRepository)`) instead of creating them internally

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
minimal layout in the `go-project-layout` skill and grows into layers when a second adapter or a
non-trivial business rule appears — do not scaffold empty layers.

```
internal/
├── domain/           # Business entities and rules
│   ├── user.go       # User entity
│   └── order.go      # Order entity
├── application/      # Use cases / services
│   ├── ports.go      # Outbound ports: the interfaces the use cases consume
│   ├── user_service.go
│   └── order_service.go
├── adapters/
│   ├── http/         # HTTP handlers (inbound)
│   ├── grpc/         # gRPC handlers (inbound)
│   └── postgres/     # Database implementation (outbound)
└── infrastructure/
    ├── config/       # Configuration loading
    └── wire/         # Dependency injection
```

## Ports and Adapters

### Inbound Ports (Driving)

Entry points that invoke domain logic:
- HTTP/REST handlers
- gRPC services
- CLI commands
- Message queue consumers

### Outbound Ports (Driven)

Interfaces the use cases need, declared by the package that calls them (normally `application`;
`domain` only when a domain service itself makes the call):
- Repository interfaces
- External API clients
- Message publishers
- Cache interfaces

```go
// Outbound port (application/ports.go — declared where it is consumed)
type UserRepository interface {
    Save(ctx context.Context, user *User) error
    FindByID(ctx context.Context, id string) (*User, error)
}

// Adapter implements the port
type PostgresUserRepository struct {
    db *sql.DB
}

func (r *PostgresUserRepository) Save(ctx context.Context, user *User) error {
    // PostgreSQL-specific implementation
}
```

## Best Practices

- Keep the domain layer pure; no framework dependencies
- Use dependency injection for all external dependencies
- Define interfaces in the package that uses them; adapters import the port, never the reverse
- Test domain logic without infrastructure
- Use constructor functions to enforce required dependencies
- Avoid circular dependencies between packages

## Anti-Patterns to Avoid

- **Anemic Domain Model**: Domain objects with only getters/setters, no behavior
- **Big Ball of Mud**: No clear boundaries between concerns
- **God Package**: One package that does everything
- **Leaky Abstractions**: Infrastructure details bleeding into domain
- **Circular Dependencies**: Packages that import each other

---

## Architecture Checklist

Before finalizing design decisions, verify:

- [ ] Domain layer has no external dependencies
- [ ] Dependencies point inward (toward domain)
- [ ] Interfaces defined at consumer, not provider
- [ ] Each package has a single, clear responsibility
- [ ] External systems accessed through interfaces
- [ ] Business logic testable without infrastructure
- [ ] No circular dependencies between packages
- [ ] Constructor functions enforce required dependencies
