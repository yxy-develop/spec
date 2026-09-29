# Decision 0020: Ownership surface, and structs that move

- Status: Accepted — **experimental**. The direction is the provisional
  surface the author authorized on 2026-09-29 (R-7 of the decision register
  in the control repository, `plans/decisions/README.md`), from the study of
  TASK-20260926-040; it is reviewed by the author at the gate of 0.2, with
  the choices of the study's first step that this decision relies on
  (below).
- Date: 2026-09-29
- Origin: the author's decision on Q3 (2026-09-26): a struct **moves by
  default**, a copy is **explicit**, and the choice is never made by
  physical size; the provisional surface R-7 (2026-09-29); tasks
  TASK-20260926-040 (the study) and TASK-20260926-078 (this part).
- Spec: `semantics.md` §4.2 [OWN-1]–[OWN-8]; [TY-1], [TY-6], [REF-5],
  [STRUCT-6], [STRUCT-9], [ENUM-3], [DECL-1], [LOOP-4], [ORD-4], §12;
  `syntax.md` §3 (`param`, `postfix`), [GR-1], §4; decision 0012 item 2.

## Context

Until now every value of the language was copied: declaring, assigning,
passing, returning and storing a struct copied it whole ([STRUCT-6]), and
use-after-move could not happen ([REF-5]). The lowering kept that meaning
while avoiding copies nobody can observe (implementation decision 0012),
but the language had no way to say that a value is given away, no single
owner, and so no ground for owned resources (the owned string, OPEN #7;
files; allocation), destructors or exclusive borrows (OPEN #6).

The author decided Q3 on 2026-09-26: structs move by default, a copy is
explicit, never decided by size. The study of TASK-20260926-040
(`plans/evidence/TASK-20260926-040/options.md`, version 2, and
`deliverable.md`, the metrics and the table of decisions) compared two
ownership surfaces over 110 programs (95 of the suite and 15 added):
**(A)** the Rust family, with references in types and lifetimes inferred,
and **(C)** values, `inout` and limited borrows, with no reference types
for structs and no lifetimes written. On the 105 programs rewritten under
both (the paired set), (A) added 221 tokens of borrowing syntax against 70
for (C) and changed 511 lines against 270; the explicit copies were 72
against 63 and the moves 86 against 76; the errors exposed before the
fixes were nearly the same (use after move 28 against 26, maybe-moved 2
and 2, in loops 3 and 3); (C) needed judgment in 8 programs against 5 (a
view kept in a struct, a returned reference to an element), and agents
rewriting under the rules agreed with them in 17 of 21 programs under (C)
against 14 of 21 under (A). The study also measured the spellings of the
explicit copy (S0 no operation, S1 `e.copy()`, S2 `copy(e)`, S3 `copy e`),
the copyable class (K0 none, K1 opt-in, K2 structural, K3 opt-out), arrays
against structs (R1–R3), unwinding at the C boundary (U0–U2), allocation
((a), (b), (d)) and running out of memory (O1–O3). The deliverable decides
nothing; R-7 is the author's authorization to adopt the surface below as
experimental.

## Decision

### 1. The direction (R-7)

| Question | Choice | What it means | Where it is normative |
|---|---|---|---|
| Ownership surface | **(C)** | values with a single owner; a parameter borrows its argument, `inout` borrows it exclusively, `take` receives it; the views `&[T]`, `&mut [T]` and `&str` stay second class, with a lexical duration; no reference types for structs, no lifetimes written | this decision, for what the language has today (part 2); `inout`, `&mut [T]`, views with a lexical duration and the exclusivity of borrows: TASK-20260926-042 |
| Explicit copy | **S1**, `e.copy()` | a postfix operation; `copy` stays usable as a name (study: 0 fields named `copy` in the suite, 7 uses of `copy` as a local that S2 and S3 would break) | now, [OWN-6] |
| Copyable class | **K0** | no struct and no enum with data copies implicitly; `Option` and `Result` copy only when every type argument copies (the step-1 choice 7 below) | now, [OWN-1] |
| Arrays | **R2** | a whole array is copied only explicitly, `a.copy()`; `b := a` stays refused; an array is never moved as a whole | now, [TY-6], [OWN-6] |
| Unwinding at the C boundary | **U1** | an unwind that reaches a Yxy frame from foreign code aborts the process there, with a trap report | with destructors, TASK-20260926-042 (OPEN #45) |
| Allocation | **(d)** | the layers `core`, `alloc` and `std`; allocation is what `alloc` gives, not an effect or an allocator argument | TASK-20260926-044 |
| Running out of memory | **O2** | the simple form of an allocating operation aborts; a `try_` family returns `Result` | TASK-20260926-044 (and TASK-20260926-043 for the owned string) |

### 2. What is normative now

What the language has today (structs, enums with data, `Option`,
`Result`, arrays, parameters, `match`, loops; no heap, no owned string, no
`inout`) follows the rules of (C) that apply without `inout`
(`semantics.md` §4.2):

1. **Classes** ([OWN-1]): the copy types are `bool`, the integers, the
   floats, `()`, enums without data, `Option`/`Result` whose type arguments
   are all copy types, `&[T]`, `&str` and `Console`; every struct, every
   enum with a variant that holds data, `Option`/`Result` that hold one, and
   every array are move types. Never by size, never by target.
2. **Moving uses** ([OWN-2]): the value of a declaration, the right side of
   an assignment, an argument of a `take` parameter, the operand of
   `return`, the error of `require`, a field of a struct literal, a value
   of a variant, of `Some`, `Ok`, `Err`, an element of an array literal, the
   value of `[v; N]`, the value of a `match` arm, the operand of `?`. Not
   moving: `_ := p`, reads of fields of copy types, `.len`, comparisons,
   `&a`, an argument of a parameter without `take`, the subject of a
   `match`, the operand of `.copy()`. Moves are not written at the use.
3. **Parameters** ([OWN-3]): `p: T` of a move type borrows the argument for
   the call and never moves it out; `take p: T` receives the value. `take`
   is a word only before a parameter's name, and only before a parameter
   whose type moves.
4. **Moved places** ([OWN-4]): a use after a move on every path, and a use
   after a move on some paths only (maybe-moved, in a branch or in an
   earlier iteration of a loop), are errors; partial moves of fields of a
   variable that owns its value; a whole assignment makes a moved variable
   usable again, and assigning a field of a moved one is an error.
5. **Borrowed places** ([OWN-5]): a parameter without `take`, an element of
   an array or slice, the variable of a `for` over elements, and a binding
   of a `match` on one of those are never moved out. A binding of a `match`
   on a place is that part of the place: moving it moves the part (the
   whole place when a variant's value is on the way), so a use of the
   subject after it is an error; that is (C)'s rule C5 (a `match` consumes
   its subject only where the subject is not used after), written as a rule
   on the binding.
6. **The explicit copy** ([OWN-6]): `x.copy()`, for a value of a move type
   only (every component of every such type copies in this version); `.copy()`
   of a copy type is an error.
7. **Arguments are lent** ([OWN-7]): a place given to a parameter without
   `take` is borrowed from its evaluation until the call returns (no longer
   a snapshot taken when it is evaluated); a later argument that assigns or
   moves that place, or one that overlaps it, is an error.
8. **Nothing silent, no size** ([OWN-8]): the compiler inserts no copy to
   make a use valid, and the fix its diagnostics suggest is `.copy()`, as a
   suggestion.

The compiler checks rules 4, 5 and 7 with a flow analysis on its MIR (the
compiler repository, `compiler/src/moves.rs`, codes E0360–E0363), and
refuses `.copy()` of a copy type (E0364) and `take` on a copy type (E0365).
The generated code of a valid program is unchanged in meaning: a move is
carried out by copying bytes where the lowering does not avoid it, and the
tools report every such copy with its size, the explicit copies apart
(reason `explicit` in `yxy inspect --json`).

### 3. Choices of the study's first step this decision relies on

`deliverable.md` lists the choices that the rules of `options.md` fixed in
the study's first step, as choices of the author. This decision relies on
these; they are reviewed by the author at the gate of 0.2:

| # (deliverable) | Choice | Here |
|---|---|---|
| 1 | A maybe-moved place is an error, with no flag at run time | [OWN-4], E0361 |
| 2 | A partial move only of a local that owns its value (a variable or a `take` parameter); a move out of an element of an array or through a borrow is an error | [OWN-4], [OWN-5], E0362 |
| 3 | `_ := p` does not move `p` | [OWN-2], [DECL-1] |
| 4 | `[e; N]` stays valid for move types, as a written form of copying (not RP-strict) | [OWN-2], [ORD-4] |
| 5 | `Console` (and, later, the other capabilities) is a copy type | [OWN-1] |
| 6 | Enums with data follow structs: move types, never copied implicitly under K0 | [OWN-1], [ENUM-3] |
| 7 | `Option` and `Result` copy when every type argument copies; an enum of the program with data always moves | [OWN-1] |
| 8 | Arguments are no longer snapshots: an argument is borrowed or moved when it is evaluated | [OWN-7], E0363 |
| 11 | Moves are not written at the use (MV0) | [OWN-2] |
| 12 | `take` is a contextual word, written at the parameter and not at the call | [OWN-3] |
| 15 | A `match` consumes its subject only where the subject is not used after (here: moving a binding moves its part of the subject) | [OWN-5] |

Choice 9 (hoisting a copy before a call whose arguments conflict) needs
`inout`: here the conflict of [OWN-7] is fixed by `.copy()` on the earlier
argument, in place. Choice 10 (a parameter becomes `take` when the body
moves it) was the study's procedure for rewriting, not a rule: the
migration of the compiler's suite writes `.copy()` where a value is used
again, the fix the diagnostics suggest, and changes no signature. Choices
13 and 14 (views with a lexical duration; an element of a list is a place)
and 16 (a trap and a foreign unwind run no destructor) have nothing to act
on before TASK-20260926-042. Choice 17 (names taken by the spelling) is
answered by S1, which takes none.

Two choices are this task's, under R-7, and are reviewed with the others:

- **`.copy()` exists only for the move types**, and `.copy()` of a copy type
  is an error with a mechanical fix that removes it (E0364). The study's
  rule B6 allowed `COPY` of a copy type as never needed; refusing it keeps
  one spelling for one meaning: `.copy()` in a program marks exactly the
  copies that cost more than a value of a copy type.
- **`take` exists only before a parameter whose type moves** (E0365, with a
  mechanical fix that removes it).

## Alternatives

- **(A), the Rust family** (references `&T` and `&mut T` in types, with
  lifetimes inferred and written where they cannot be): more syntax in
  every program that borrows (221 tokens against 70 in the study, 511
  lines changed against 270), references in fields and results, and more
  divergent rewrites by agents (14 of 21 agreeing against 17 of 21). It
  answers the four programs that (C) needed judgment for (a view kept in a
  struct, a reference returned into a container), which (C) rewrites with an
  index, a copy or a function.
- **(B), second-class references only**: contained in (C) as far as
  parameters go; not measured on its own.
- **(D), a collector**: excluded by M-2 (nothing inserted in silence).
- **S0, no copy operation** (a literal that reads every part, a `match` per
  variant): 747 components and 187 constructors written in (A), 577 and 178
  in (C), and impossible for a struct of another package with a private
  field. **S2 `copy(e)`**: 3 tokens per copy against S1's 4, but takes the
  name `copy` from 7 uses in the suite. **S3 `copy e`**: 1 token per copy,
  but a reserved word that breaks the same 7 uses and every field named
  `copy`, and a new prefix operator.
- **K1 (opt-in `copy struct`)**, **K2 (structural)** and **K3 (opt-out)**:
  under K2 and K3, 404 of the study's 415 declarations (131 without its
  two generated programs) would copy implicitly, against Q3; K1 is
  compatible with Q3 and, under (C), would take the copies from 63 down to
  3 with 25 markers at most, and can be added later without breaking a
  program (a type that copies accepts every program that its moving form
  accepts).
- **R1 (arrays move as structs)** and **R3 (no whole-array copy, as
  before)**: R2 keeps `b := a` refused and gives the explicit copy that R3
  lacked.
- **Maybe-moved allowed with a flag at run time** (drop flags): a hidden
  cost and a hidden state; an error keeps the program's meaning in its text.
- **Snapshots kept for arguments** (the meaning before this decision): an
  argument copied when a later argument changes the place is a copy the
  program did not write; [OWN-7] refuses the program instead, and the fix
  writes the copy.

## What could change it

- The author's review at the gate of 0.2: every choice of part 1, the
  choices of part 3, and the two choices of this task.
- TASK-20260926-042 (MIR-1): `inout`, `&mut [T]`, the exclusivity of
  borrows beyond arguments (a place changed while a view or a binding of it
  lives), destructors and their order, U1, and the generalization of these
  rules to the types with an owner.
- TASK-20260926-043 and -044: the owned string, whose copy is deep and
  allocates (`.copy()` then allocates, under (d) and O2), and the layers.
- K1, if programs show many copies of small structs that a marker would
  remove: an addition, compatible with every valid program.
- Evidence that `take` at the parameter only (MV0) hides moves from
  readers or agents: `move` at the use (MV1, MV2) was measured and left
  out.
