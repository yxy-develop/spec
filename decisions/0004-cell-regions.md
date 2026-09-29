# Decision 0004: Cell regions

- Status: Experimental — plans/decisions/README.md, L-0004; item 3 is the author's
  decision A1 (2026-09-26), itself experimental
- Date: 2026-09-25; item 3 revised 2026-09-26
- Spec: `semantics.md` [CELL-1]–[CELL-8], `syntax.md` §5
- Errata (2026-09-26): wording only; no rule changed. Item 4 and [CELL-6]
  now say that the calls made by declarations of `@ctrl` before a false guard
  have already run; the compiler always behaved so. The errata does not
  change A1 (the author's decision of 2026-09-26), which changed item 3 as a
  rule.

## Context

The author's central pattern is `@ctrl <- @eval -> @effect/@out`: control on
the left, central production in the middle, propagation on the right. The
bootstrap instructions define the intent of each region, forbid inventing
runtime mechanisms to make the construct "different", and allow it to reduce to
conventional control flow.

## Decision

1. A cell is a whole function body. Regions appear in the order `@ctrl`,
   `<- @eval`, `-> @effect` (any number), `-> @out`; each is optional; `@ctrl`
   takes no arrow, `@eval` takes `<-`, `@effect`/`@out` take `->`. Wrong or
   missing arrows and wrong order are errors. Compact and multi-line forms are
   the same grammar.
2. Execution is sequential in lexical order. There are no threads per region,
   no speculation and no reordering of operations that can fail or have
   effects. A failure prevents every later step and undoes nothing.
3. *(author decision A1, 2026-09-26; experimental)* Each region admits only
   some statements, at any depth: `require` in `@ctrl` (pre-validation) and in
   `@eval` (post-validation, after the computation it checks); `return` only
   in `@out`; `@ctrl` holds declarations and `require`. An early exit before
   `@out` uses a false `require` or `?`. A `require` may appear anywhere in
   `@eval`; since statements run in lexical order, it validates what the
   statements before it computed, and a false one prevents the rest of
   `@eval`, every effect step and the output. This gives the regions of a
   cell a checked meaning that tools can rely on **for that function**:
   validations run in `@ctrl` or `@eval`, before every effect step. It is not
   a property of the whole program: a function whose body is not a cell, or a
   helper called from `@eval`, is not restricted.
4. Effects are not restricted by region: `@effect` grants no permission and
   `@eval` may call operations whose effects are declared. Consequently a
   declaration in `@ctrl` may call a function with effects, and that call
   happens before a guard that follows it; a false guard prevents the
   evaluation and every later step, not the effects already performed
   ([CELL-6]).

## Alternatives

- No restriction per region except that `@out` transfers control (proposed in
  the independent specification draft of 2026-09-25): regions become labels
  with an order. Simpler, and avoids inventing rules the author did not state;
  but the labels then carry no checked information.
- `require` only in `@ctrl` (the provisional rule before A1). Post-validation
  then needed a helper function called from `@eval`, which the region rule
  could not see. The author chose post-validation in `@eval`.
- For the place of `require` inside `@eval` (A1 says "after the
  computation"):
  - only as a trailing block, after the last other statement of `@eval`:
    forbids validating an intermediate value before a computation that uses
    it (a divisor computed in `@eval`), which forces a helper function again;
  - only when its condition mentions a name declared in `@eval`: a syntactic
    stand-in for "post", easy to defeat and a rule the author did not state;
  - a separate post-condition region or construct: adds a concept.

  Rejected in favour of "anywhere in `@eval`": lexical order already makes
  every `require` of `@eval` a check of what was computed before it.
- Calls in `@ctrl` restricted to functions without effects (`effects {}`), so
  that a false guard prevents every effect, as the author's acceptance text of
  the bootstrap instructions (§18) reads. Not adopted: a new rule that would
  force validations that read something (an environment, a file) into `@eval`
  with `?`. [CELL-6] states instead what happens; the alternative stays open
  (`OPEN.md` #1).
- Cells anywhere (inside `if`, as expressions). Deferred: needs a defined way
  for a value to leave a nested cell that is not `return`.

## What could change it

The author's further reading of item 3; programs where the restriction forces
unnatural code; effects in `@ctrl` that surprise readers; the design of `when`
and of nested cells.
