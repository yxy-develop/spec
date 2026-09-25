# Decision 0008: Typed holes

- Status: Accepted — experimental
- Date: 2026-09-25
- Spec: `semantics.md` [HOLE-1]

## Decision

The single typed-hole marker is `$` or `$name`, in expression position. The
compiler reports it with the type expected at that position; `check` and
`build` reject any program that contains one.

## Alternatives

- `@hole` (independent draft): fits the `@` family of structural markers.
  Deferred: `@` currently marks regions only, and `$name` lets several holes be
  told apart.
- `_` or `?`: rejected, they already mean wildcard/discard and propagation.
