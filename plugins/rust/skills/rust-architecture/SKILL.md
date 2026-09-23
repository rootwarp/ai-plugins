---
description: 'Rust architecture rules: SOLID principles, layering, dependency direction, trait boundaries. Use when designing modules or crates, choosing where logic belongs, or reviewing whether a change respects the layers.'
name: rust-architecture
---

# Architecture

Follow [Clean
Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) and
[Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/) principles adapted
for Rust's ownership model and trait system.

## SOLID Principles

- **Single Responsibility** — each module does one thing well; split structs with multiple reasons
  to change; keep business logic separate from infrastructure (logging, HTTP, database)
- **Open/Closed** — extend by adding new implementations of existing traits (e.g. a new
  `Notifier`), not by modifying existing code; prefer composition over trait inheritance
- **Liskov Substitution** — implementations must honor the trait's documented contract (e.g. a
  `Reader` returns `Ok(0)` at EOF and never reads past `buf.len()`)
- **Interface Segregation** — keep traits small (1-3 methods); define them at the consumer, not
  the provider; compose via supertraits when needed
  (`trait UserRepository: UserReader + UserWriter {}` instead of one fat trait)
- **Dependency Inversion** — depend on trait bounds (`UserService<R: UserRepository>`) or trait
  objects (`Box<dyn UserRepository>`); inject dependencies through constructors instead of
  creating them internally

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

## Recommended Module Structure

This is the target for a service with real domain logic. A small tool or library starts with the
minimal layout in the `rust-project-layout` skill and grows into layers when a second adapter or
a non-trivial business rule appears — do not scaffold empty layers.

```
src/
├── lib.rs            # Public API, re-exports
├── domain/           # Business entities and rules
│   ├── mod.rs
│   ├── user.rs       # User entity
│   └── order.rs      # Order entity
├── application/      # Use cases / services
│   ├── mod.rs
│   ├── ports.rs      # Outbound ports: the traits the use cases consume
│   ├── user_service.rs
│   └── order_service.rs
├── adapters/
│   ├── mod.rs
│   ├── http/         # HTTP handlers (inbound)
│   ├── grpc/         # gRPC handlers (inbound)
│   └── postgres/     # Database implementation (outbound)
└── infrastructure/
    ├── mod.rs
    ├── config.rs     # Configuration loading
    └── di.rs         # Dependency injection setup
```

## Ports and Adapters

### Inbound Ports (Driving)

Entry points that invoke domain logic:
- HTTP/REST handlers (axum, actix-web)
- gRPC services (tonic)
- CLI commands (clap)
- Message queue consumers

### Outbound Ports (Driven)

Traits the use cases need, declared by the module that calls them (normally `application`;
`domain` only when a domain service itself makes the call):
- Repository traits
- External API clients
- Message publishers
- Cache traits

```rust
// Outbound port (application/ports.rs — declared where it is consumed)
#[async_trait]
pub trait UserRepository: Send + Sync {
    async fn save(&self, user: &User) -> Result<(), Error>;
    async fn find_by_id(&self, id: &UserId) -> Result<Option<User>, Error>;
}

// Adapter implements the port
pub struct PostgresUserRepository {
    pool: Pool<Postgres>,
}

#[async_trait]
impl UserRepository for PostgresUserRepository {
    async fn save(&self, user: &User) -> Result<(), Error> {
        // PostgreSQL-specific implementation
    }
}
```

## Best Practices

- Keep the domain layer pure; no framework dependencies
- Use dependency injection via generics or trait objects
- Define traits in the module that uses them; adapters import the port, never the reverse
- Test domain logic without infrastructure
- Use constructor functions to enforce required dependencies
- Avoid circular dependencies between modules

## Anti-Patterns to Avoid

- **Anemic Domain Model**: Domain objects with only getters/setters, no behavior
- **Big Ball of Mud**: No clear boundaries between concerns
- **God Module**: One module that does everything
- **Leaky Abstractions**: Infrastructure details bleeding into domain
- **Circular Dependencies**: Modules that import each other

---

## Architecture Checklist

Before finalizing design decisions, verify:

- [ ] Domain layer has no external dependencies
- [ ] Dependencies point inward (toward domain)
- [ ] Traits defined at consumer, not provider
- [ ] Each module has a single, clear responsibility
- [ ] External systems accessed through traits
- [ ] Business logic testable without infrastructure
- [ ] No circular dependencies between modules
- [ ] Constructor functions enforce required dependencies
