# Decision 0004: Cell regions

- Status: Accepted — experimental; item 3 is explicitly provisional
- Date: 2026-09-25
- Spec: `semantics.md` [CELL-1]–[CELL-8], `syntax.md` §5

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
3. *(provisional)* Each region admits only some statements: `require` only in
   `@ctrl`; `return` only in `@out`; `@ctrl` holds declarations and `require`.
   An early exit before `@out` uses `?`. This gives the regions a checked
   meaning that tools can rely on ("validation happens in `@ctrl`").
4. Effects are not restricted by region: `@effect` grants no permission and
   `@eval` may call operations whose effects are declared.

## Alternatives

- No restriction per region except that `@out` transfers control (proposed in
  the independent specification draft of 2026-09-25): regions become labels
  with an order. Simpler, and avoids inventing rules the author did not state;
  but the labels then carry no checked information.
- Cells anywhere (inside `if`, as expressions). Deferred: needs a defined way
  for a value to leave a nested cell that is not `return`.

## What could change it

The author's reading of item 3; programs where the restriction forces
unnatural code; the design of `when` and of nested cells.
