# Decision 0008: Typed holes

- Status: Experimental — plans/decisions/README.md, L-0008
- Date: 2026-09-25
- Spec: `semantics.md` [HOLE-1]
- Errata (2026-09-26): wording only; no rule changed. The decision promised
  more than the compiler does: the reach of [HOLE-1] in this version is
  stated below and in the specification.
- Errata (2026-09-29): wording only; no rule changed. The reach below
  followed the compiler's partial facts (compiler requirement R756).

## Decision

The single typed-hole marker is `$` or `$name`, in expression position. The
compiler reports it with the type expected at that position when the context
gives one, and says when it does not; `check` and `build` reject any program
that contains one.

In this version (errata of 2026-09-26 and 2026-09-29) the diagnostic gives
the expected type in its text (a note of E0313). When every error of a
program lies in the body of a function, holes included, `inspect --json`
still gives the facts of its other functions: partial facts that list each
function with an error or a hole as a gap, with its holes and each hole's
expected type as data (`expected_type`, `null` when the context gives none;
compiler requirement R756). The names in scope at a hole are future work.

## Alternatives

- `@hole` (independent draft): fits the `@` family of structural markers.
  Deferred: `@` currently marks regions only, and `$name` lets several holes be
  told apart.
- `_` or `?`: rejected, they already mean wildcard/discard and propagation.
