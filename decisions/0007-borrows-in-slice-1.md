# Decision 0007: Borrows in the first subset

- Status: Accepted — experimental
- Date: 2026-09-25
- Spec: `semantics.md` §4

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
