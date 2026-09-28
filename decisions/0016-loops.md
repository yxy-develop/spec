# Decision 0016: Loops: `loop`, `for`, `break` and `continue`

- Status: Accepted — **experimental** (proposed with task
  TASK-20260926-033 of the master plan of 2026-09-26, Phase 5; open to the
  author's review)
- Date: 2026-09-28
- Spec: `syntax.md` [LEX-5], [LEX-6], [NL-8], [GR-6], [GR-8], §3 (`loop`,
  `for`, `break`, `continue`); `semantics.md` [TY-4], [DECL-3], [FN-2],
  [FN-3], [CELL-2], [LOOP-1]–[LOOP-7], §12; `OPEN.md` #47, #49
- Author requirements it follows (register of the control repository): M-2
  (nothing silent: no default integer type chosen without a rule that says
  so, no copy of an array, no wrapping at the end of a range), P-2 (traceable
  costs: iteration has no hidden check or allocation)

## Context

Until this decision `while` was the only loop, and `for`, `loop`, `break`,
`continue` and `in` were reserved words rejected as "not supported yet", with
a note to use `while`. The author's first real programs write
`for i in 0..10 { … }`, and leaving a loop from its middle needed a flag in the
condition.

Loop exits also touch the rule that decides whether a statement can be
followed by another: before this decision a `while` whose condition was the
literal `true` counted as never ending, which decided [FN-2] (a function that
returns a value must not reach its end), unreachable statements and the type of
a `match` arm block ([FN-3]). With `break`, `while true { if c { break } }`
ends, so that rule would accept a function that reaches its end without a
value, and the generated code would reach the instruction it places there
only when the end cannot be reached. The master plan (Phase 5, rule of
containment of the architecture audit §6.5 item 2) asks that every
construct that exits a loop fix that rule in the same change, with tests in
the three engines, and that no `unreachable` of the generated code be
reachable through `break`.

There are no generics yet (OPEN #5), so there is no iterator protocol to
build `for` on, and slices only view immutable arrays ([REF-1]).

## Decision

1. **Words and syntax** ([LEX-5], §3 of `syntax.md`). `break`, `continue`,
   `for`, `in` and `loop` become keywords. Three statements and two
   one-word statements:

   ```yxy
   loop { … }                       // until a `break`
   for i in 0..n { … }              // 0, 1, …, n − 1
   for i in 1..=n { … }             // 1, 2, …, n
   for i: u64 in 0..10 { … }        // the variable's type written
   for x in s { … }                 // each element of a slice or named array
   for _ in 0..3 { … }              // three times, no variable
   break                            // leave the innermost loop
   continue                         // next iteration of the innermost loop
   ```

   `..` and `..=` exist only in the head of `for`; they bind more loosely
   than every operator (`0..n + 1` ends at `n + 1`), and each bound is a head
   ([GR-6]: a struct literal there is parenthesized). The `{` of `loop` and
   `for` is on the line of the keyword or the head ([NL-8]). `break` and
   `continue` take nothing after them.
2. **Loops** ([LOOP-1], [LOOP-2]). `while`, `loop` and `for` are statements.
   `break` leaves the innermost loop whose body contains it; `continue` ends
   the current iteration of that loop (`while` evaluates its condition again,
   `for` takes the next value, `loop` runs its body again). A `break` or
   `continue` in a `match` arm leaves the loop around the `match`. The
   condition of `while` and the head of `for` are not part of the body: a
   `break` there belongs to the loop around it. Outside the body of every
   loop, `break` and `continue` are errors.
3. **Integer ranges** ([LOOP-3]). `a..b` is `a`, `a + 1`, …, `b − 1`, empty
   when `a ≥ b`; `a..=b` ends at `b` included, empty when `a > b`. Both bounds
   have one integer type, taken from each other, as the operands of `+`
   ([TY-4]), or from the annotation of the variable; when neither gives one
   (`0..10`), the type is `usize`, the type of a position ([REF-4]), which
   [TY-4] now says. Both bounds are evaluated once, left to right, before the
   first iteration: changing what they read inside the body does not change
   the iterations. Iterating never overflows and never traps: the loop stops
   after its last value instead of computing the next one, so
   `for i: u8 in 250..=255` runs six times.
4. **Elements** ([LOOP-4]). `for x in s` where `s` is a slice (`&[T]`: a
   parameter, a local, `&a`, `t.bytes`) or names an array local, which is the
   same as `&a` and so requires what `&a` requires: an array that is not
   `mut` ([REF-1]). `s` is evaluated once; each iteration binds `x` to a copy
   of the next element, first to last. A slice views an immutable array, so
   nothing can change its elements or its length during the loop: iteration
   needs no check at run time and has none. A `mut` array is iterated by
   index (`for i in 0..a.len { … a[i] … }`). Text is not iterated ([TEXT-7]);
   its bytes are (`for b in t.bytes`).
5. **The loop variable** ([LOOP-1]). It is declared in the scope of the body
   (not visible after the loop) and immutable: a new value at each
   iteration, which the body cannot assign. It follows [DECL-3] (no
   shadowing). `_` iterates without a variable.
6. **Ends of statements** ([LOOP-5]). A statement *goes on* when execution
   can continue after it. `return`, `break` and `continue` never go on. An
   `if` goes on when one of its branches does (a missing `else` does). A
   `while` whose condition is the literal `true`, and every `loop`, go on only
   when a `break` in their body leaves them; any other `while`, and every
   `for`, may go on. A block goes on when all its statements do; an
   expression goes on when it produces a value. This one rule decides:
   - [FN-2]: a function whose type is not `()` must not go on at its end
     (`while true { if c { break } }` at the end of a function that returns
     `u8` is an error);
   - unreachable statements: those after a statement that does not go on;
   - [FN-3] and the type of a `match` arm block: a block that does not go on
     has no value; one that goes on has type `()` (an arm
     `{ while true { if c { break } } }` has type `()`).

   Without labels, the loop a `break` leaves is found in the text, so the
   rule is exact for the statements of this version; it is checked on the
   typed program, where the three engines (native at `-O0`, `-O2` and the
   reference evaluator) read it.
7. **Cells, regions and effects** ([LOOP-6], [LOOP-7]). Loops are statements
   of `<- @eval` and `-> @effect`, at any depth, like `while`; `break` and
   `continue` follow the rules of the region of their loop. They leave a
   loop, never a region or the cell: an early exit before `@out` is still
   only a false `require` or `?` ([CELL-2]). `@ctrl` and `@out` hold no
   loops. A loop has no effect of its own; running for ever is not an effect
   ([EFF-1]); iterating has no check, so no trap and no site ([TRAP-3]).

## Alternatives considered

- **Which loops.** Only `while` and `loop` (no `for`): rejected, counting is
  the most common loop and writing it with `while` spreads the index over
  three statements. C's `for (init; cond; step)`: rejected, `;` is not a
  terminator ([LEX-16]) and the step is where off-by-one errors hide. Go's one
  `for` for every loop (`for cond {}`, `for {}`): rejected, `while` exists,
  and one form per intent reads better; `loop` says "until `break`" where
  `while true` says it through a literal.
- **What `for` iterates.** An iterator protocol (Rust's `IntoIterator`,
  Swift's `Sequence`): needs traits and generics (OPEN #5); later, it can
  take over the two forms of this version without changing their meaning.
  Ranges as values of their own type (`r := 0..3`): a type is needed only to
  store or pass a range; left open (OPEN #47, #49), since accepting it later
  changes no valid program.
- **The type of `0..10`.** An error asking for a type (E0303, like
  `x := 0`): rejected, `for i in 0..10` is the first loop people write, and
  its bounds are positions far more often than anything else. Inference from
  the uses of the variable in the body: the checker types by expected type,
  without inference variables. A default of the program's choosing (`i32`,
  `int`): no such type exists. Chosen: `usize` as the context of the bounds
  when nothing else gives one, written into [TY-4] as an explicit rule, not a
  default of literals; `for i: u64 in 0..10` or a typed bound chooses
  another.
- **Only half-open ranges.** Rejected: a range that ends at the largest
  value of its type (`0..=255` for `u8`) cannot be written half-open.
- **The end of a range at the type's maximum.** A trap when the step
  overflows: rejected, `for i: u8 in 0..=255` would trap after its last
  iteration. Wrapping: rejected, the same loop would never end. Chosen: the
  loop never computes a value after its last.
- **Changes during iteration.** Copying the array before iterating (Go):
  rejected, arrays are never copied silently ([TY-6]). Reading a `mut` array
  live, so that the loop sees the body's changes: rejected, it is surprising,
  and it would not stay sound for collections that grow. A borrow rule that
  lets the body change what it does not iterate (Rust): needs ownership
  (OPEN #6). Chosen: only what a slice may view is iterated by element, so
  no change is possible and no check is needed.
- **Labels.** Rust's `'outer:`: `'` starts a character literal, refused
  (E0109). Go's `outer:` with `break outer`: `break x` would then be a label
  or a value, depending on names in scope. Zig's `:outer`. Chosen: none in
  this version; a nested exit uses a flag or a `return` from a function.
  Adding labels later only accepts more programs.
- **`break` with a value from `loop`** (Rust, `x := loop { … break v }`).
  Deferred: blocks have no value in this version (`if` is a statement,
  [GR-3], and a `match` arm block has type `()` or none), and `loop` would
  become the first expression made of statements. A value that a loop finds
  is assigned to a `mut` variable declared before it, then `break`. Making
  `loop` an expression later changes no valid program (as for `if`, [GR-3]).
- **`break` in the condition of `while`.** Rust refuses it without a label.
  Chosen: the lexical rule — the condition is not the body, so a `break`
  there leaves the loop around it — which needs no special case.
- **The rule for loop exits.** Keeping "`while true` never ends": wrong with
  `break` (the problem above). An analysis on a control-flow graph (the
  MIR-0 of the architecture audit, §6): not yet; the structural rule is exact
  for this version and the MIR-0 verifier will check it.

## What could change it

- The MIR-0 and its verifier (master plan, Phase 6): a disagreement between
  the structural rule and the graph.
- Generics (OPEN #5): an iterator protocol for `for`; ranges as values.
- Ownership (OPEN #6): iterating collections that can change, with a
  borrow rule instead of immutability.
- Programs that need labels or `break` with a value often enough; the
  measurements of `for` against `while` in the benchmarks.
- The author's review of the type of an unannotated range (`usize`).
