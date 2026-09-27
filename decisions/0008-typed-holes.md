# Decision 0008: Typed holes

- Status: Accepted — experimental
- Date: 2026-09-25
- Spec: `semantics.md` [HOLE-1]
- Errata (2026-09-26): wording only; no rule changed. The decision promised
  more than the compiler does: the reach of [HOLE-1] in this version is
  stated below and in the specification.

## Decision

The single typed-hole marker is `$` or `$name`, in expression position. The
compiler reports it with the type expected at that position when the context
gives one, and says when it does not; `check` and `build` reject any program
that contains one.

In this version (errata of 2026-09-26) the expected type is part of the
diagnostic's text (a note of E0313), not a field of its own, and a program
that still contains a hole has no facts (`inspect --json`) for its other
functions either (tested by `json_outputs_are_valid_and_stdout_only` in the
compiler's conformance suite). A structured description of holes for tools
and agents — the expected type as data, the names in scope, facts of the rest
of the program — is future work.

## Alternatives

- `@hole` (independent draft): fits the `@` family of structural markers.
  Deferred: `@` currently marks regions only, and `$name` lets several holes be
  told apart.
- `_` or `?`: rejected, they already mean wildcard/discard and propagation.
