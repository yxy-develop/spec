# Decision 0002: Declarations and mutability

- Status: Accepted — experimental (proposed by the orchestrator under the
  author's bootstrap instructions, §5; open to the author's review)
- Date: 2026-09-25
- Spec: `semantics.md` [DECL-1]–[DECL-5]

## Context

The bootstrap instructions propose `:=` for local declarations and ask that its
mutability not be inherited from Go: declarations should be immutable, with
mutability marked explicitly, and `let`, `var` and `:=` must not become
synonyms.

## Decision

- `name := value` declares an immutable local; `mut name := value` a mutable
  one. With a type: `name: T := value`, `mut name: T := value`.
- `=` only assigns to an existing `mut` variable or to an element of a `mut`
  array. `:=` always declares; `=` never declares.
- `_ := value` discards a value explicitly; unused non-`()` values are errors.
- No shadowing inside a function; a local cannot reuse an item's name.
- `let` and `var` are not declarations; using them is an error that points to
  `:=`.

## Alternatives

- `name: T = value` for annotated declarations (Go-like `var x T = v`). Rejected
  for now: one symbol for "declare" (`:=`) and one for "assign" (`=`) is easier
  to read and to check, for humans and agents.
- Allowing shadowing (Rust). Rejected for now: a name then means one thing per
  function, which lowers the context needed to read a region.

## What could change it

A reading study or agent benchmark showing more errors with `name: T := value`
than with `name: T = value`; real programs where the ban on shadowing forces
awkward names.
