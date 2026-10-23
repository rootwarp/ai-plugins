---
description: 'Go code style: formatting, names, code organization, error strings, and comments. Use when you write or review Go source code.'
name: go-code-style
---

# Code Style

The baseline is [Effective Go](https://go.dev/doc/effective_go), the [Google Go Style
Guide](https://google.github.io/styleguide/go/best-practices.html), and the [Uber Go Style
Guide](https://github.com/uber-go/guide/blob/master/style.md). The rules below are the decisions
that this project adds to them.

## Formatting and Names

- Format the code with `gofmt` or `goimports` (see the `go-fmt` skill). Keep lines to about 100
  columns.
- Do not put the type in a name. Write `users`, not `usersMap`. Use short names for short scopes.
- Do not use generic package names (`utils`, `common`, `helpers`). Give a package a name for
  what it provides.

## Organization

- Put imports in three groups in this sequence: the standard library, external modules, and
  the packages of this module.
- Make identifiers unexported by default. Export only the identifiers that callers need.
- Use few packages, and make each package cohesive. Divide packages by responsibility (see the
  `go-architecture` skill). Do not divide them by kind (`models`, `handlers`, `types`).
- You do not have to put each concept in a separate file. Where practical, put functions in the
  sequence that they are called.

## Interfaces

- Define an interface in the package that uses it. Keep interfaces small.
- Pass an interface by value. Do not use a pointer to an interface.
- Make sure at compile time that a type implements an interface:
  `var _ Interface = (*Implementation)(nil)`.

## Errors

- Return errors. Use `panic` only for a state that the program cannot recover from.
- Handle each error one time. Log it or return it, but do not do both.
- Add the context that the underlying error does not have. Wrap with `%w` only when callers
  must use `errors.Is` or `errors.As`. A wrapped error becomes part of your API.
- Use sentinel errors or custom error types for errors that callers make decisions on.

## Functions and APIs

- Make `context.Context` the first parameter of each function that does I/O.
- If a function has more than two or three parameters of the same type, a caller can put them
  in the wrong sequence. In that case, use a struct parameter
  (`Send(ctx, Message{To: ..., Subject: ...})`).
- Use guard clauses that return early. They keep the main path unindented.
- At API boundaries, copy slices and maps if the caller or the callee can change shared state.

## Performance

- Use `strconv`, not `fmt`, for conversions.
- When you know the size, allocate slices and maps with that size.
- In hot paths, do not convert between `string` and `[]byte` again and again.

## Comments and Messages

Write doc comments, error messages, and log messages in ASD-STE100 Simplified Technical English
(STE):

- Write short sentences. Use a maximum of 20 words in an instruction and 25 words in a
  description.
- Use the active voice and the simple present tense.
- Use one term for one concept. Do not use synonyms for the same concept.
- Do not write a noun cluster of more than three words.
- Do not use contractions, semicolons, or Latin abbreviations.

The Go conventions have priority over STE. A doc comment starts with the name of the
identifier. An error string starts with a lowercase letter and has no final period.

The `go-concurrency` skill gives the concurrency rules. The `go-tdd` skill gives the test rules.
