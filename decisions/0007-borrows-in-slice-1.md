# Decision 0007: Borrows in the first subset

- Status: Experimental — plans/decisions/README.md, L-0007
- Date: 2026-09-25
- Spec: `semantics.md` §4
- Amendment (2026-09-28, experimental): decision 0015 adds a second
  reference type, borrowed text `&str`, whose values view constant data and
  may therefore also be returned (`semantics.md` [TEXT-6]); what this
  decision says of `&[T]` does not change.

## Decision

- The only reference type is the read-only slice `&[T]`.
- `&a` borrows an immutable array local. Borrowing a `mut` array is an error,
  so a slice never observes a mutation.
- Slices can be parameters and locals, but cannot be returned or stored in
  arrays, `Option` or `Result`; therefore no slice outlives its array.

This is a restriction by construction, not a borrow checker. It is stated as
such so that no stronger guarantee is claimed than the one checked.

## What could change it

Ownership of resources (0.2): unique ownership, moves, exclusive mutable
borrows, borrows of `mut` values with checked lifetimes, and references that
may be returned when they point into a parameter.
