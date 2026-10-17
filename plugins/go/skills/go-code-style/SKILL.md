---
description: 'Go code style: formatting, naming, code organization, error strings. Use when writing or reviewing Go source.'
name: go-code-style
---

# Code Style

Baseline: [Effective Go](https://go.dev/doc/effective_go), the [Google Go Style
Guide](https://google.github.io/styleguide/go/best-practices.html), and the [Uber Go Style
Guide](https://github.com/uber-go/guide/blob/master/style.md). The rules below are the choices
this project makes on top of them.

## Formatting and Naming

- Format with `gofmt` / `goimports` (see the `go-fmt` skill); keep lines around 100 columns.
- Don't encode the type in a name (`users`, not `usersMap`). Short names for short scopes.
- No generic package names (`utils`, `common`, `helpers`). Name a package for what it provides.

## Organization

- Import groups: stdlib, external, then the module's own packages.
- Unexported by default; export only what callers need.
- Prefer fewer, cohesive packages. Split by responsibility (see the `go-architecture` skill),
  never by kind (`models`, `handlers`, `types`).
- One concept per file isn't required. Order functions by invocation where practical.

## Interfaces

- Define interfaces where they're consumed and keep them small. Pass them by value, not as
  pointers to interfaces.
- Assert compliance at compile time: `var _ Interface = (*Implementation)(nil)`.

## Errors

- Return errors; panic only on truly unrecoverable states.
- Handle an error exactly once. Either log it or return it, not both.
- Add context the underlying error lacks. Wrap with `%w` only when callers need `errors.Is` /
  `errors.As`, since wrapping makes the inner error part of your API.
- Use sentinel errors or custom types for errors callers branch on.

## Functions and APIs

- `context.Context` is the first parameter of any function that does I/O.
- Past two or three same-typed parameters that are easy to swap, take a struct
  (`Send(ctx, Message{To: ..., Subject: ...})`).
- Return early with guard clauses to keep the happy path unindented.
- Copy slices and maps at API boundaries when the caller or callee could mutate shared state.

## Performance

- `strconv` over `fmt` for conversions.
- Pre-allocate slices and maps when the size is known.
- Avoid repeated `string` ↔ `[]byte` conversions in hot paths.

Concurrency rules live in the `go-concurrency` skill, and tests in `go-tdd`.
