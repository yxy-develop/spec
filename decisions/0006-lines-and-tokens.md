# Decision 0006: Newlines, terminators and tokens

- Status: Experimental — plans/decisions/README.md, L-0006
- Date: 2026-09-25
- Spec: `syntax.md` §1, §2

## Decision

- Newlines terminate statements; `;` is not a terminator (it only appears in
  `[T; N]`). Newlines are ignored inside `( )` and `[ ]`, after a binary
  operator, before `else`, between the parts of a signature, and around items,
  variants, arms and region headers.
- A line that starts with an operator does not continue the previous line.
- Tokens use maximal munch, so `<-` is always the region arrow (`a<-b` is an
  error). Control tokens are ASCII. Identifiers are ASCII in this version.
- `@io` is rejected, with a mechanical fix to `@out`.
- *(clarification, 2026-09-26)* There is no compound assignment (`+=` and the
  other `op=` forms) in this version: an assignment is written
  `x = x + value`. The grammar never had it; `semantics.md` §12 now lists it
  among the constructs rejected with a diagnostic.

## Alternatives

- Optional `;` (independent draft): two ways to end a statement. Rejected for
  now in favour of one textual form per concept.
- Continuation when a line starts with an operator: ambiguous with prefix
  operators that may start statements. Rejected.
- Unicode identifiers (UAX #31 with NFC). Deferred: needs a policy for
  confusable characters, normalization and symbol mangling.
- Compound assignment (C, Go, Rust): shorter, but a second spelling of an
  assignment, and `a[f()] += 1` evaluates its target once where the long form
  evaluates it twice. Not in this version; usage evidence could bring it.
