# Decision 0018: Constants, and warnings of certain traps

- Status: Accepted — **experimental** (proposed with task TASK-20260926-035
  of the master plan of 2026-09-26, Phase 5, after the architecture audit of
  2026-09-26: T19, option (2), a research recommendation, and §21 C2,
  option (1), with correction 4 of its legend; revised after its review of
  2026-09-28; open to the author's review at the gate of 0.1, below)
- Date: 2026-09-28
- Spec: `syntax.md` [LEX-5], [LEX-6], §3 (`const_decl`, the length of
  `[T; N]` and of `[value; N]`); `semantics.md` [PRG-2], the table of §2,
  §3.1 [CONST-1]–[CONST-7], [TGT-2], §12; `modules.md` [INIT-2], [VIS-1];
  `OPEN.md` #50
- Author requirements it follows (register of the control repository):
  L-0003 (integer arithmetic with exact traps: a constant is computed by the
  same rules as the program), P-2 (traceable costs: a constant costs nothing
  at run time, and a check whose operands are constants is still a check),
  and the direction of the author's bootstrap instructions on evaluation at
  compile time (§7: explicit limits, no textual metaprogramming)

## Context

Until this decision `const` was a reserved word (E0900), and the length of
an array type or of `[value; N]` was an integer literal: a size used in
several places was written again at each, and a program could not give a
size, a mask or a table bound a name. The audit (T19) weighed four options:
(1) no constants; (2) constants of literals, operators and other constants,
without calls; (3) `const fn` run by the reference evaluator under limits of
steps and memory; (4) unrestricted evaluation at compile time. (4) turns the
checker into an interpreter and hides the expansion that the architecture
asks to show. (3) reuses the evaluator's semantics, but the evaluator has
no limit of steps today, and a mistake in it would give the same wrong
constant to the three engines (native at `-O0`, `-O2` and the evaluator),
which then could not disagree. (2) covers sizes and masks without running
functions and meets [INIT-2] with little machinery; it is the audit's
recommendation for this phase, with the limit that [INIT-2] asks for from
the start and a reading of [TGT-2] written for the results of constants.

The same audit (§9, §21 C2) asks what to do with a trap that the compiler
can see is certain. An error would make validity depend on the target
(`65536 * 65536` in `usize` traps only on 32 bits), against [TGT-2], and,
with constants propagated through variables, would reject the conformance
suite (`tests/run/traps.yxy`, whose traps are on immutable locals, in
branches chosen by the input: correction 4 of the audit's legend). The
recommendation is a warning, which never changes validity, and a policy for
warnings, which the compiler had only for the backend's E0702.

## Decision

1. **Declaration** ([CONST-1], [LEX-5], `syntax.md` §3). `const` becomes a
   keyword. `const NAME: T := value` declares a constant, an item of its
   package; `pub const` makes it visible to other packages, which name it
   `pkg.NAME` ([VIS-1]). The type is written, an integer type or `bool`:
   literals have no type of their own ([TY-4]), and a written type keeps
   the value's type visible where it is declared. A constant is declared at
   package level only.
2. **Constant expressions** ([CONST-2]). The value is made of integer
   literals, `true`, `false`, names of constants (`N`, `pkg.N`), the
   operators of §4, grouping parentheses and `widen(x)` of a constant
   expression: the conversion of [CONV-1], which never traps, keeps the
   value and has the same rule on every target, so that a size of `usize`
   gives a wider constant (`const BYTES: u64 := widen(SLOTS) * 8`, computed
   in `u64`). Nothing else — no other call (not `checked_convert`, not
   `checked_add`: there is no `const fn`), no variable, no field, `match`,
   text or array. It is typed by the rules of function bodies.
3. **Evaluation** ([CONST-3]). The compiler computes every constant of the
   program, used or not, before code generation, by the rules of §6 on the
   target's data model: left to right, `&&` and `||` evaluating their right
   operand only when needed. A name of a constant stands for its value. An
   operation that would trap makes the program invalid: an error at that
   operation that names the kind of failure. Nothing runs: no effects, no
   package code ([INIT-2]). The compiler computes the value with its own
   arithmetic, written from §6, not with the reference evaluator's, and the
   conformance suite compares each constant with the same operations done
   at run time by the three engines — the check that T19 asks for.
4. **Limits** ([CONST-4], [INIT-2]). A constant that reads itself, directly
   or through others, is an error. The *steps* of a constant are its
   literals, operators, `widen` and names, and a name of a constant also
   counts the steps of that constant: the size of the expression with every
   constant written out, which is the work an evaluator that remembers
   nothing would do, so the limit does not depend on how the compiler
   computes. A constant of more than 1 000 000 steps is an error, and so is
   an array length or count of more, counted the same way: an expression is
   refused as a length exactly when it is refused as a constant. Both are
   found from the names before anything is computed, the same on every
   target, and the check always ends. The constant expressions of function
   bodies, computed only for the warnings of (7), are not limited: they read
   the values of constants already limited, their own size bounds the work,
   and a warning decides nothing.
5. **Array lengths** ([CONST-5]). The length of `[T; N]` and the count of
   `[value; N]` are constant expressions of type `usize`, under the limit
   of (4).
6. **Targets** ([CONST-6], [TGT-2]). A constant's value, and so its
   validity, may depend on the target only through the range and width of
   `usize` and `isize`, exactly as a literal's does: `65536 * 65536` in
   `usize` is 2^32 on a 64-bit target and an error on a 32-bit one. A
   constant whose value reads no `usize` or `isize` value has the same
   value on every target; one that reads one may differ, whatever its own
   type (a comparison of type `bool`, a `widen` to `u64`).
   [TGT-2] says so in its text. With (3), every constant computed, a
   `pub const` that is an error on 32-bit targets only makes every program
   that imports its package invalid there, even one that never reads it;
   [CONST-6] and OPEN #50 say so.
7. **Certain traps** ([CONST-7]). An operation of a function body whose
   operands are constant expressions, and whose computation traps, does not
   make the program invalid: the program traps when it evaluates the
   operation, as before. The compiler reports a **warning** there, one per
   largest constant expression, at the operation where the program would
   report its trap. An operation with a variable operand is not computed at
   compile time, even when it always traps (`x / 0`), and gets no warning:
   a warning is never wrong, and no program of the suite gets one. A
   warning never makes a program invalid and never changes what it does
   ([TGT-2]); it may depend on the target. The compiler reports warnings as
   it reports errors, with a code and a severity, in its human and
   structured output, from the same diagnostic.
8. **With the floats and the enums with data** (decisions 0019 and 0017,
   decided alongside this one). Floats have no constants: a constant of type
   `f32` or `f64` is refused by its type, and a float literal is not a
   constant expression ([CONST-1], [CONST-2]). An integer constant
   expression inside `to_float`, `fma` or the values of a variant with data
   is one like any other, and gets the warning of (7) when it traps; the
   operand of `truncate_to_int` is a float, never a constant expression, so
   its trap is never computed at compile time. A constant is not a pattern
   in any of the forms of decision 0017: a name binds, and `pkg.N`, or a
   constant as the name of a struct pattern, is an error ([CONST-1]). A
   constant is typed, so a bound of a `for` range that is a constant gives
   the range its type ([LOOP-3]): `for i in 0..N` with `N` of type `u8`
   iterates `u8`.

## Alternatives considered

- **No constants** (T19 (1)): leaves sizes written again at each use.
- **Calls in constant expressions.** None, as the first text of this
  decision had it: a size of `usize` could not give a constant of a wider
  type, which had to be written again as a second constant with the same
  number. `const fn` (T19 (3)): below. Chosen: `widen` alone: it never
  traps and its rule does not depend on the target ([CONV-1]), so it adds
  a step and no case of failure to (3).
- **`const fn` run by the reference evaluator** (T19 (3)): needs limits of
  steps and memory in the evaluator, and the three engines would share one
  computation of every constant; kept as a later experiment (OPEN #50),
  with the values of constants compared at run time as the suite does now.
- **Unrestricted evaluation at compile time** (T19 (4), types as values):
  rejected for now; it makes the checker an interpreter and hides the
  expansion.
- **The type of a constant.** Inferred from the value, with untyped
  constants that take the type of their use (Go): a second kind of integer
  value, and a constant whose type changes with each use; against [TY-4],
  which gives literals no default type. Chosen: written, as in Rust.
- **Types of constants.** Text (`&str`: string literals are constant data,
  [TEXT-1]), enums, structs, arrays, floats: each needs rules of its own
  (equality, layout in the facts, rounding for floats, decision 0019);
  integers and `bool` cover sizes, masks and flags. Adding the others later
  only accepts more programs; `&str` is the cheapest next step (OPEN #50).
- **Constants inside functions.** An immutable local already names a value
  there; a local constant would be needed only for a length computed from
  names of the function, which (5) allows from package constants. Later,
  if programs need it; accepting it then changes no valid program.
- **Evaluating only the constants a program uses** (lazily): the validity
  of a declaration would depend on its uses. Chosen: every constant, with
  its consequence for a `pub const` valid on 64-bit targets only, in (6).
- **The measure of the limit.** Steps of an evaluator that remembers the
  values it computed (the work of this compiler): it would change with the
  implementation, and a program valid for one compiler could be invalid for
  another. A limit on the depth of the chains of constants: not a measure of
  work. No limit, since option (2) always ends: [INIT-2] asks for explicit
  limits, and the step measure is the one a later `const fn` would need.
  Chosen: the size of the expression written out, 10^6 steps; chains of
  10^5 constants check in a fraction of a second.
- **A failed constant as a warning**, with a trap at run time: a constant
  is also a length, needed while the program is compiled; it has no value
  to run with. Chosen: an error, like a literal that does not fit.
- **Certain traps** (C2). (2) an error for expressions of literals in
  fixed-width types and a warning elsewhere: independent of the target,
  but one more rule and a class of programs refused that run correctly
  until the branch is taken. (3) an error always: against [TGT-2], and the
  suite would be rejected (correction 4). Chosen: (1), a warning.
- **What a certain trap is.** Also an operation that traps whatever its
  variable operand is (a division by the constant 0, a shift by a constant
  at least the width, a constant index of an array whose length is known):
  never false either, and on the audit's `sure.yxy` it finds four traps
  instead of one. Its warnings on the conformance suite
  (`tests/run/traps.yxy`, `one << 64`) would be true: the traps there are
  deliberate. With constants propagated through immutable locals: it would
  warn on the traps of `tests/run/traps.yxy`, which the input may never
  reach. Both stay experiments (OPEN #50). The rule chosen gives no warning
  at all on the conformance suite and the examples, so that a warning, when
  one appears, is news, and it is the rule the compiler also uses for
  constants, so one sentence explains both.
- **A separate series of codes for warnings** (`W…`): the compiler already
  reports a warning with an `E` code and the severity `warning` (E0702),
  and tools filter by severity; a series of its own later would rename
  documented codes. Chosen: the same convention. Levels of warnings and the
  warnings of the packages of other modules: OPEN #50, before the first
  dependency is published.

## What could change it

- The author's review, at the gate of 0.1.
- The experiment of `const fn` run by the reference evaluator under limits
  (T19 (3)), with the values of constants compared at run time.
- Floats (decision 0019): constants of `f32` and `f64`, with the rounding of
  their literals and operations.
- Programs that need constants of text, enums or structs, constants inside
  functions, constants in patterns (`match x { N => … }`), or calls other
  than `widen` in constant expressions.
- Programs that reach the limit of steps, or a measure of a later evaluator
  that needs another one.
- Warnings in real use: the broader rules of certain traps, levels of
  warnings (allowed, denied), and warnings of the packages of other modules.

## For the author's review at the gate of 0.1

The review of this decision (2026-09-28) put the choices below to the
author. Each is in force, experimental until the author reviews it at the
gate of 0.1:

1. **The type of a constant** — always written ([CONST-1]). Inferring it
   from the value, or untyped constants that take the type of their use
   (Go), stay alternatives (above).
2. **Where, and of what types** — at package level only, of the integer
   types and `bool`, in 0.1. `&str` constants are the cheapest next step
   (OPEN #50); accepting them, or local constants, later changes no valid
   program.
3. **Calls** — `widen` is allowed ((2)); no other call, and `const fn`
   stays an experiment (OPEN #50).
4. **The scope of the warning** — the rule of (7), for constant
   expressions only. Its reason is that it gives no warning on the
   conformance suite, not that the broader rule's warnings would be false:
   they would be true ("What a certain trap is", above).
5. **Every constant computed** — kept ((3)); its consequence for a
   `pub const` valid on 64-bit targets only is in (6), [CONST-6] and
   OPEN #50.
6. **The code of a warning** — an `E` code with the severity `warning`, as
   E0702 (above); levels of warnings and the warnings of the packages of
   other modules are decided before the first dependency is published
   (OPEN #50).
