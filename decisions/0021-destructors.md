# Decision 0021: Destructors and deterministic destruction

- Status: Accepted — **experimental**, under the provisional surface R-7
  (the decision register of the control repository,
  `plans/decisions/README.md`); reviewed by the author at the gate of 0.2
  with decision 0020.
- Date: 2026-09-30
- Origin: the author's requirement M-1 (single ownership, an exclusive
  mutable borrow, **deterministic destruction**) and E-2 (the effect check
  is transitive through calls, recursion, callbacks and destructors); rule
  B4 of the study of TASK-20260926-040 (`plans/evidence/TASK-20260926-040/
  options.md` §2); TASK-20260926-042 (MIR-1).
- Spec: `semantics.md` §4.3 [DROP-1]–[DROP-6]; [CELL-8], [TRAP-1],
  [ABI-3] (c), [OWN-1], [OWN-6], §12; `syntax.md` §3 (`fn_decl`).

## Context

Decision 0020 gave every value a single owner and made structs, enums with
data and arrays move. Its rule B4 of the study says what ownership is for:
an owned value whose type has a destructor is destroyed exactly once, at
the end of the scope of its owner, in reverse order of declaration, on
every path that leaves the scope, and never after a move; a trap destroys
nothing, and the effects of a destructor are effects of the functions that
destroy the value. The study leaves open how a type gets a destructor: the
owned types of its §11 (`text.String`, `fs.File`, lists) are the library's,
and the heap that they need comes with TASK-20260926-044 (allocation, the
layers `core`, `alloc` and `std`, decision 0020 part 1).

TASK-20260926-042 must test destruction before that heap exists: on every
exit (`require`, `?`, `return` in an arm, `break`, the end of a body), in
the three engines of the compiler (native code at `-O0` and `-O2`, the
reference evaluator), with the effects of a destructor in the effect
check. That needs a mechanism a program can use today, and the owned
string and the other owned types will be built on the same one.

## Decision

1. **A destructor is a function of the struct's package**, written
   `drop fn name(v: S) effects { … } { … }` ([DROP-1]): one parameter of a
   struct `S` declared in the same package, without `take` or `inout` (the
   destructor reads the value; it is destroyed after it, field by field),
   no result, not `pub`, not `extern` or `export`, at most one for each
   struct. It is never called by name: it runs when a value of `S` is
   destroyed. `drop` is a word only before `fn` at the start of an item, so
   a name `drop` stays usable. Only structs have destructors in this
   version; an enum, `Option`, `Result` or array is destroyed through the
   values it holds.
2. **Which types are destroyed** ([DROP-2]): a struct with a destructor,
   and a struct, an enum with data, an `Option`, a `Result` or an array that
   holds one. Destroying a value runs the struct's destructor, then
   destroys its fields in declaration order; the values of the variant an
   enum holds, in order; the payload of `Some`, `Ok` or `Err`; the elements
   of an array from the first to the last. Every type of the language
   before this decision has no destructor, and destroying it does nothing:
   no program changes meaning, and its generated code is the same.
3. **When** ([DROP-3]): B4 of the study, made exact. The owner of a value
   destroys it: a variable at the end of the block that declares it, the
   last declared first, on every path that leaves the block (the end of
   the block, `return`, a false `require`, a `?` that returns, `break`,
   `continue`); a `take` parameter at the end of its function, after the
   variables of the body; the old value of an assigned place, after the new
   value is computed and before it is stored; the new value of `_ := e`, at
   once. A value moved out is not destroyed; the parts of a struct without
   a destructor that were not moved are. A value moved on some paths only,
   where it would be destroyed, is an error (E0377 of the compiler), like
   maybe-moved ([OWN-4]): nothing records at run time whether it moved. A
   trap destroys nothing ([TRAP-1]), and neither does an unwind from
   foreign code ([ABI-3] (c)).
4. **No copy, no part alone** ([DROP-4]): a value whose type has a
   destructor, or holds one, is never copied: `.copy()` and `[v; N]` of it
   are errors (the copy and the value would both be destroyed). A field of
   a struct that has a destructor never moves out alone (the destructor
   reads the whole value); a struct without a destructor gives a field away
   as before ([OWN-4]).
5. **Effects** ([DROP-5], E-2): the effects of a destructor are effects of
   every function in which a value of its type, or of a type that holds
   it, is destroyed, which declares them as for a call ([EFF-3]); its
   callers declare them in turn. The compiler reports the missing effect
   where the value is destroyed, with the destructor and the effect.
6. **Every value has an owner** ([DROP-6]): a new value whose type has a
   destructor (a call's result, a literal) is in a position that moves it
   ([OWN-2]), or is discarded with `_ :=`. In a position that does not own
   it (the argument of a parameter without `take`, the struct of a field
   read, the subject of a `match`) it is an error, fixed by declaring it
   first. An exit that may leave an expression (a `?`, or a `return`,
   `break`, `continue` or false `require` in a block of a `match` arm
   inside the expression) while a value with a destructor that the
   expression made or that was moved into it waits for its owner (an
   earlier argument of a call that has not run, a field of a literal being
   built) is decided by where that value is:
   - a value the expression **made** is in no place: the exit is an error,
     fixed by declaring the value first;
   - a place **moved into** the expression (an argument of a `take`
     parameter, a field of a struct literal, an element of an array
     literal, a value of a variant) gives its value when the expression
     takes it: when the call runs, when the value is built. An exit before
     that leaves the value in the place, and the place's owner destroys it
     where it destroys its values on that path ([DROP-3]), in its order.
     The place stays moved for its uses ([OWN-4]): a use after such an
     exit is a use after a move, and a destruction that some paths reach
     with the value kept and others with it moved is an error, as for any
     value moved on some paths only. A place given a new value between
     the move and the exit (in a `match` arm of a later operand) no longer
     holds the moved value, which would be held by nothing: that exit is
     an error.

   So every value made or moved into an expression is destroyed exactly
   once on every exit, every destruction is the end of an owner's scope,
   an assignment or `_ :=`, and no destruction needs a flag or a hidden
   temporary. *(Amended 2026-10-04 by the review of
   TASK-20260926-042: the first text spoke only of the values the
   expression made, and a place moved into it was destroyed by no one on
   such an exit.)*
7. **What is not destroyed at run time is not guessed**: the compiler
   decides every destruction from the flow of the function (the analysis
   of moves), the same on every target; the facts of `yxy inspect` list
   each destruction, where it happens, its reason and the destructors it
   may run, beside the copies that carry out moves (P-2).

## Alternatives

- **A `drop` operation of a trait or interface** (Rust's `Drop`): the
  language has no traits (OPEN #5); a function of the struct's package,
  marked by one contextual word, needs none, and a later trait can be
  defined to mean exactly it.
- **A destructor that takes the value** (`drop fn f(take v: S)`): the
  destructor could then move fields out, which leaves the rest to destroy
  in a partial state, and its own end would destroy `v` again unless that
  rule had an exception. Reading the value and destroying the fields after
  it keeps one rule for every owner.
- **Destructors of enums, or a destructor as a field attribute**: nothing in
  the programs of the study needs them; an enum holding a struct with a
  destructor is destroyed through it. Adding them later accepts more
  programs.
- **Drop flags** (maybe-moved values destroyed where a flag at run time
  says so): a hidden state and a hidden cost, rejected for the same reason
  as maybe-moved values in decision 0020 (item 1 of its part 3).
- **Temporaries destroyed at the end of their statement** (C++ and Rust):
  valid programs of item 6 would then include `look(make(1))`, at the cost
  of hidden temporaries with their own order of destruction; declaring the
  value first costs one line and states the order. Allowing it later
  accepts more programs.
- **Refusing an exit while a place moved into the expression waits** (the
  error of item 6 for moved places too, as for the values the expression
  made): the place still holds the value until the expression takes it,
  since nothing reads a moved place again and a move hands the value over
  only when the call runs or the literal is stored, so its owner can
  destroy it with no flag and no hidden temporary, in the order of
  [DROP-3]. Refusing would make the common `pair(g, fails(n)?)` declare
  the later operand first for no gain in clarity, while a value the
  expression made has no place to stay in and is still refused. The
  place stays moved for its uses, so the analysis of moves of every
  expression that completes is unchanged.
- **Unwinding through Yxy frames, running destructors**: rejected by the
  study (§8) and by R-7 (U1): a trap and a foreign unwind end the process
  and run no destructor.
- **Copying a value with a destructor by a user-defined copy**: a later
  addition (the owned string's copy is deep and allocates, TASK-20260926-043
  and -044); until then `.copy()` of such a value is refused.

## What could change it

- The author's review at the gate of 0.2, with decision 0020.
- The owned types of TASK-20260926-043 and -044: they are structs of the
  library with destructors under this decision; their copy (deep, with
  allocation) will need `.copy()` for types with a destructor.
- Programs that need temporaries with destructors in positions that do not
  own them, or destructors of enums: additions that accept more programs.
