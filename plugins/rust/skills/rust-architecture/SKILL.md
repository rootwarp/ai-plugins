---
description: 'Rust architecture rules: SOLID principles, layers, dependency direction, and trait boundaries. Use when you design modules or crates, when you decide where logic goes, or when you review if a change obeys the layers.'
name: rust-architecture
---

# Architecture

This skill applies Clean Architecture and Hexagonal Architecture with the traits and the
ownership model of Rust.

## Layers

| Layer | Responsibility | Can depend on |
|-------|----------------|---------------|
| Domain | Entities and business rules | Nothing (no framework crates) |
| Application | Use cases, orchestration, outbound port traits | Domain |
| Adapters | HTTP, gRPC, and CLI handlers (inbound), DB and API clients (outbound) | Application, Domain |
| Infrastructure | Config loading, wiring, runtime setup | All layers |

Dependencies point only inward. Modules never import each other in a cycle.

This structure is the target for a service with real domain logic. A small tool or library
starts with the minimal layout in `rust-project-layout`. Add layers when a second adapter or a
non-trivial business rule appears. Do not make empty layers.

```text
src/
├── lib.rs                # public API, re-exports
├── domain.rs, domain/    # user.rs, order.rs
├── application.rs, application/
│   ├── ports.rs          # outbound ports: the traits the use cases consume
│   └── user_service.rs
├── adapters.rs, adapters/
│   ├── http.rs           # inbound
│   └── postgres.rs       # outbound
└── infrastructure.rs, infrastructure/
    ├── config.rs
    └── di.rs             # wiring
```

Module files use the `name.rs` + `name/` form (see `rust-project-layout`).

## Trait Boundaries

- Declare a port trait in the module that uses it. This is usually `application`. Use
  `domain` only when a domain service itself makes the call. Adapters import the port. The port
  never imports an adapter.
- Keep each trait to 1-3 methods. Combine traits with supertraits
  (`trait UserRepository: UserReader + UserWriter {}`). Do not make one trait larger and larger.
- Inject dependencies through constructors. By default, use generic bounds
  (`UserService<R: UserRepository>`). Use trait objects (`Arc<dyn UserRepository>`) when you
  need selection at runtime or heterogeneous collections.
- Each implementation obeys the documented contract of its trait, so you can substitute any
  implementation for another. For example, a `Reader` returns `Ok(0)` at EOF and never returns
  more than `buf.len()`.
- To extend behavior, write a new implementation of a trait that exists. Do not change the
  callers.

## Async Ports

Native `async fn` in traits (Rust 1.75+) does not need a macro for generic injection. Declare
methods as `fn ... -> impl Future<Output = ...> + Send`. Then callers can spawn the futures on a
multi-threaded runtime. Implementations can still use plain `async fn`:

```rust
// application/ports.rs
pub trait UserRepository: Send + Sync {
    fn find_by_id(&self, id: &UserId) -> impl Future<Output = Result<Option<User>, Error>> + Send;
}

// adapters/postgres.rs
impl UserRepository for PostgresUserRepository {
    async fn find_by_id(&self, id: &UserId) -> Result<Option<User>, Error> {
        todo!("SELECT ... via self.pool")
    }
}
```

Native async traits are not dyn-compatible. If you need `Box<dyn …>` or `Arc<dyn …>`, put
`#[async_trait::async_trait]` on the trait and on its impls.
