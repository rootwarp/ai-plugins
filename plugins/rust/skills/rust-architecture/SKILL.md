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
  `Notifier`), not by modifying existing code; prefer composing structs over deep trait
  hierarchies
- **Liskov Substitution** — implementations must honor the trait's documented contract (e.g. a
  `Reader` returns `Ok(0)` at EOF and never reads past `buf.len()`)
- **Interface Segregation** — keep traits small (1-3 methods); define them at the consumer, not
  the provider; combine small traits via supertrait bounds rather than growing one large trait
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
├── domain.rs         # Declares the domain submodules
├── domain/           # Business entities and rules
│   ├── user.rs       # User entity
│   └── order.rs      # Order entity
├── application.rs
├── application/      # Use cases / services
│   ├── ports.rs      # Outbound ports: the traits the use cases consume
│   ├── user_service.rs
│   └── order_service.rs
├── adapters.rs
├── adapters/
│   ├── http.rs       # HTTP handlers (inbound)
│   ├── grpc.rs       # gRPC handlers (inbound)
│   └── postgres.rs   # Database implementation (outbound)
├── infrastructure.rs
└── infrastructure/
    ├── config.rs     # Configuration loading
    └── di.rs         # Dependency injection setup
```

Module files use the `name.rs` + `name/` form, as in the `rust-project-layout` skill.

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
// Outbound port (application/ports.rs — declared where it is consumed).
// Native `async fn` in traits (Rust 1.75+); no macro needed for generic injection.
pub trait UserRepository: Send + Sync {
    fn save(&self, user: &User) -> impl Future<Output = Result<(), Error>> + Send;
    fn find_by_id(&self, id: &UserId) -> impl Future<Output = Result<Option<User>, Error>> + Send;
}

// Adapter implements the port; `async fn` satisfies the `impl Future + Send` signature
pub struct PostgresUserRepository {
    pool: PgPool,
}

impl UserRepository for PostgresUserRepository {
    async fn save(&self, user: &User) -> Result<(), Error> {
        todo!("INSERT ... via self.pool")
    }

    async fn find_by_id(&self, id: &UserId) -> Result<Option<User>, Error> {
        todo!("SELECT ... via self.pool")
    }
}

// Use case consumes the port through a generic bound
pub struct UserService<R: UserRepository> {
    repo: R,
}
```

Writing the trait methods as `fn ... -> impl Future<Output = ...> + Send` (instead of plain
`async fn`) lets callers spawn the futures on a multi-threaded runtime. Native async traits are
not dyn-compatible: if you need `Box<dyn UserRepository>` or `Arc<dyn UserRepository>`, annotate
the trait and impls with `#[async_trait::async_trait]` instead.

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
