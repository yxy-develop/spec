# Decision 0002: Declarations and mutability

- Status: Experimental — plans/decisions/README.md, L-0002 (proposed under the
  author's bootstrap instructions, §5; open to the author's review)
- Date: 2026-09-25; amended 2026-10-04
- Spec: `semantics.md` [DECL-1]–[DECL-5]; `syntax.md` [LEX-6], [LEX-7]
  (amendment of 2026-10-04)

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

## Amendment (2026-10-04): `let` and `var` are reserved

Until this amendment `let` and `var` were identifiers ([LEX-7]): only a
statement that starts with `let name` or `var name` was refused, and both
were valid names of locals, parameters, fields and variants. The audit of
the specification (§D.1, S1) and the gate of 0.1 recommended reserving them
before the opening of the language, since reserving them after it breaks the
programs that use them as names (OPEN #48). The author's answer to the gate
(2026-10-04, item [LEX-7] of the gate's request; TASK-20261004-082) accepts
it:

- `let` and `var` join the reserved words of [LEX-6]. Unlike the other
  reserved words they do not mean "not supported yet": using either as a
  name (a local, a parameter, a field, a variant, after `.`) is an error with
  a note pointing to `:=` (the compiler's E0900, as for every reserved word
  used as a name). There is no mechanical fix for a name: a rename changes
  every use.
- A statement that starts with `let name` or `var name` keeps its diagnostic
  and its fix to `:=` ([LEX-7]): `let x: u8 = 1` becomes `x: u8 := 1`, `var`
  becomes `mut`.
- Why: a restriction that can be relaxed later, which keeps both futures
  open, including bringing `let` back with a meaning; one spelling for
  declarations, so that `let` never reads as a name in one place and as a
  habit of another language in another.

## What could change it

A reading study or agent benchmark showing more errors with `name: T := value`
than with `name: T = value`; real programs where the ban on shadowing forces
awkward names.
