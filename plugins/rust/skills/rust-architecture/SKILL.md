---
description: 'Rust architecture rules: SOLID principles, layering, dependency direction, trait boundaries. Use when designing modules or crates, choosing where logic belongs, or reviewing whether a change respects the layers.'
name: rust-architecture
---

# Architecture

Clean / Hexagonal Architecture, expressed with Rust's traits and ownership.

## Layers

| Layer | Responsibility | May depend on |
|-------|----------------|---------------|
| Domain | Entities and business rules | Nothing (no framework crates) |
| Application | Use cases, orchestration, outbound port traits | Domain |
| Adapters | HTTP/gRPC/CLI handlers (inbound), DB/API clients (outbound) | Application, Domain |
| Infrastructure | Config loading, wiring, runtime setup | All layers |

Dependencies point inward only, and modules never import each other in a cycle.

This is the target for a service with real domain logic. A small tool or library starts with the
minimal layout in `rust-project-layout` and grows into layers when a second adapter or a
non-trivial business rule appears. Don't scaffold empty layers.

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

- Declare a port trait in the module that consumes it, normally `application` (`domain` only
  when a domain service itself makes the call). Adapters import the port, never the reverse.
- Keep traits to 1-3 methods. Combine them with supertraits
  (`trait UserRepository: UserReader + UserWriter {}`) instead of growing one fat trait.
- Inject dependencies through constructors, as generic bounds (`UserService<R: UserRepository>`)
  by default or trait objects (`Arc<dyn UserRepository>`) when you need runtime selection or
  heterogeneous collections.
- Implementations honor the trait's documented contract, so any one can be substituted (a
  `Reader` returns `Ok(0)` at EOF and never more than `buf.len()`).
- Extend behavior with a new implementation of an existing trait, not by editing callers.

## Async Ports

Native `async fn` in traits (Rust 1.75+) needs no macro for generic injection. Declare methods as
`fn ... -> impl Future<Output = ...> + Send` so callers can spawn the futures on a
multi-threaded runtime; implementations may still write plain `async fn`:

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

Native async traits are not dyn-compatible. If you need `Box<dyn …>` or `Arc<dyn …>`, annotate
the trait and its impls with `#[async_trait::async_trait]` instead.
