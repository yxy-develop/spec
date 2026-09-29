# Decision 0017: Enums with data, struct patterns and the bound of exhaustiveness

- Status: Experimental — plans/decisions/README.md, L-0017
  (proposed with task TASK-20260926-034 of the master plan of 2026-09-26,
  Phase 5; open to the author's review; the budget of the analysis (item 10),
  `..` in struct patterns (item 8), the absence of a test of a variant (item
  4) and the payloads without overlap (item 5) are reviewed by the author at
  the gate of 0.1)
- Date: 2026-09-28
- Spec: `syntax.md` §3 (`enum_decl`, `variant`, `pattern`, the note after
  the grammar), [GR-9]; `semantics.md` [TY-1], [TY-3], [DECL-6], §2.2
  [ENUM-1]–[ENUM-5], [STRUCT-7], [STRUCT-8], [STRUCT-9], [MATCH-1],
  [MATCH-4]–[MATCH-6], [NUM-5], [ABI-2], §12; `OPEN.md` #5, #32, #33, #35,
  #46
- Author requirements it follows (register of the control repository): M-2
  (nothing silent: a pattern never ignores a field or a value without
  saying so, and no layout is changed without a record), P-2 (traceable
  costs: the copies of enums with data are reported like those of structs,
  and the layout is in the facts), Y-8 (a short learning curve: one form for
  a variant's data, the form `Some(v)` already has); the author's decision on
  Q3 (structs, and so the data of enums, are copied until the ownership
  phase)

## Context

Until this decision an enum was a list of names, and a variant carrying a
value was refused ("enum variants with data are not supported yet"). The only
values with a variant and data were `Option` and `Result`, known to the
compiler. Real programs need their own: a message with its fields, a shape
with its sizes, an error with its detail. `OPEN.md` #5 left them for the
0.1/0.2 milestone.

Struct patterns (`OPEN.md` #33) were refused too: a `match` on a struct was an
error, and `P { x: 1, y }` in a pattern gave one E0900. The architecture audit
of 2026-09-26 (§5.3.1, B4 and D3) asks that they be designed together with
enum payloads, with exhaustiveness over the fields, because both make the
matrix of the exhaustiveness check wider than one column: until now every
constructor held at most one value (`Some`, `Ok`, `Err`), and the check was
linear in the arms. With constructors of several fields its worst case is
exponential in the number of columns (B7). The unit tests of TASK-20260926-032
measure it: `2^(n+1)` steps for a tuple of `n` booleans and `2n` arms. A
program must never make the compiler run away, so the check needs a bound.

The audit also records (§5 F9, C8) that `match` used to read an inactive
payload of a value in memory before testing its tag. TASK-20260926-061 fixed
it (the tag is tested before a payload is read, R122 of the compiler), and
the layout of payloads (`OPEN.md` #35) stays free only if that holds for the
payloads of every enum.

## Decision

1. **Variants with data** ([ENUM-1], `syntax.md` §3). A variant is a name, or
   a name followed by the types of the values it holds, in parentheses:

   ```yxy
   enum Shape {
       Empty
       Circle(u32)
       Rect(u32, u32)
       Framed(Size, Color)
   }
   ```

   A variant written with parentheses holds at least one value (`V()` is an
   error, with a fix that removes `()`). The types are value types other than
   arrays, as for struct fields: integers, `bool`, `()`, enums, structs,
   `Option` and `Result`; slices, text and capabilities are refused with the
   diagnostics they get as fields. Named fields (`V { a: T }`) are not
   supported: a variant with named data holds a struct, `Framed(Size)`. An
   enum still has 1 to 256 variants, variants without data and with data
   mixed in any order. An enum that holds itself, directly or through
   structs, other enums, `Option` or `Result`, would have infinite size and is
   an error, as a struct that holds itself ([STRUCT-2]).
2. **Construction** ([ENUM-2]). `Shape.Rect(3, 4)`, or `geometry.Shape.Rect(3,
   4)` for an enum of an imported package: exactly one value per type the
   variant declares, each checked against its type (from which an integer
   literal takes its type, [TY-4]), evaluated left to right, each completely,
   before the value exists; a trap or a `?` that returns in one value
   prevents the later ones ([ORD-5] for struct literals). A variant with data
   is always written with its values: `Shape.Circle` alone is an error. A
   variant without data is written without parentheses, as before,
   `Shape.Empty`, also in an enum that has variants with data.
3. **Values** ([ENUM-3]). An enum with data is a value type ([TY-1]): it is
   copied on declaration, assignment, passing, returning and storing, as
   structs are ([STRUCT-6]), until the ownership phase (the author's decision
   on Q3); its copies are reported in the facts like those of structs (P-2).
   It may be a parameter, a result, a local, a field, the data of another
   variant, an element of an array and the value of `Option` and `Result`.
4. **No equality** ([ENUM-4], [NUM-5]). `==` and `!=` stay defined for an enum
   whose variants hold no data; for an enum with a variant that holds data
   they are an error, with a note to use `match`. Structs have no equality
   either (`OPEN.md` #32), nor `Option` and `Result`. Testing which variant a
   value holds therefore takes a `match` of two arms; a form that tests a
   variant and reads only the tag (`s is Shape.Empty`) is recorded in
   `OPEN.md` #32 as a request of use. The author reviews this refusal at the
   gate of 0.1.
5. **Layout** ([ENUM-5], [STRUCT-8]; `OPEN.md` #35, in part). An enum without
   data is its one-byte tag, as before. An enum with data is laid out as
   `{ tag: u8, payload of variant 0, payload of variant 1, … }`: the tag is
   the variant's number in declaration order; the payload of a variant with
   data is laid out as a struct of its values, in order, with natural
   padding; the payload of a variant without data is empty (size 0,
   alignment 1) and takes no space. Payloads are **not** overlapped: each has
   its own offset, as `Result` keeps both of its payloads. `Option` and
   `Result` keep their layout. The layout is internal ([STRUCT-8]), may
   change, and `yxy inspect --json` reports it for each target: the size and
   alignment of the enum, the offset and size of the tag, and for each
   variant its tag, the offset and size of its payload and the offset, size
   and alignment of each value. Only the tag and the payload of the variant a
   value holds are written; a `match` reads a payload only after testing the
   tag of its variant, so an inactive payload is never read. The sum costs
   space and copies: `Wide` of the compiler's `tests/layout/enums.yxy` takes
   88 bytes on 64-bit targets, 64 with overlapped payloads (the tag, then its
   largest payload, 56 bytes at offset 8), and every copy copies the sum.
   The author reviews it at the gate of 0.1; the overlap stays tied to the
   check of the data layout and the B0 measurements (`OPEN.md` #35).
6. **The C boundary** ([ABI-2]). Enums do not cross it, with data or without,
   as before: their layout is not a stable ABI.
7. **Patterns of variants** ([MATCH-4]). `Shape.Rect(w, 0)` matches a
   variant and one pattern for each of its values, nested at any depth; a
   value that is not tested is `_`. The number of patterns is the number of
   values: `Shape.Rect(w)` and `Shape.Circle` (without its value) are errors,
   the second with a fix that writes `Shape.Circle(_)`; `Shape.Empty(x)` is an
   error. There is no rest pattern (`Shape.Rect(..)`), as for `Some(..)`. A
   name written alone for a variant of the enum being matched is an error, as
   before ([DECL-6]), with a fix that writes the variant with its enum and, for
   a variant with data, `_` for each value.
8. **Struct patterns** ([MATCH-5]; `OPEN.md` #33). `Point { x: 0, y: y }`, or
   `geometry.Point { … }` for a struct of an imported package, matches a
   struct and one pattern per field, nested at any depth. Every field is
   written exactly once, in any order, `field: pattern`, and a field that is
   not tested is `field: _`: the rule of struct literals ([STRUCT-3]). There is
   no `..` (it is an error, with a mechanical fix that writes `field: _` for
   each field it stands for) and no shorthand (`Point { x }`, the shorthand
   of literals, `OPEN.md` #40, is an error). A struct of another package with
   a private field cannot be matched with a struct pattern ([VIS-2]). A
   `match` on a struct is now valid, and its subject may be bound whole
   (`p => p.x`), as any value. Struct patterns appear only in `match` arms,
   not in declarations (`Point { x: a, y: b } := p` stays refused). The cost
   of having no `..`: a struct of another package with a private field is
   never matched with a struct pattern (E0820 of the compiler), only bound
   whole and read through its `pub` fields, which is where other languages
   need `..`; a `..` that stands only for the private fields of such a
   struct, ignoring no public field without saying so, is the extension
   recorded in `OPEN.md` #33. The author reviews it at the gate of 0.1.
9. **Exhaustiveness over fields and values** ([MATCH-1], [MATCH-2]). A
   `match` covers every value of its type, field by field and value by value,
   and names a value it does not cover (`Shape.Rect(_, false)`,
   `Point { x: _, y: true }`); an arm that earlier arms cover is an error. The
   analysis is the usefulness algorithm the compiler already used, over the
   constructors of the types: a struct has one constructor with a field per
   field, a variant one with a field per value.
10. **The bound of the analysis** ([MATCH-6]). Checking a `match` has a fixed
    budget of work, the same on every target and on every machine: when a
    `match` needs more, it is an error ("this `match` is too large to
    check"), with a note to split it; the compiler never runs away on it. The
    compiler states the budget in its own terms (the cells of pattern
    matrices it reads) in its documentation. It reads the arms through an
    index by their first constructor, so that a table of literals is checked
    in work linear in its arms. Measured with the compiler of this decision
    (its `STATUS.md`, R809): a `match` a person writes needs a small fraction
    of the budget (the largest of its test suite, 187 cells of 4 000 000);
    generated tables are accepted up to 300 000 arms of integer literals,
    155 × 155 arms on a struct of two enums (every pair an arm), 7 000 arms
    of 4-byte keys and a struct of 16 000 fields; the worst case, `2n` arms
    on a struct of `n` booleans that each test one field, is refused from
    `n = 14`. Which programs are valid thus depends on the budget and on the
    algorithm: a larger budget, or an algorithm that spends less on every
    `match`, only accepts more programs; a smaller budget after the language
    opens would refuse some.
    No minimum that every compiler must accept is fixed yet; whether the
    specification fixes one (a number of arms of a table of literals, of
    pairs of two enums) is for the author at the gate of 0.1.

## Alternatives considered

- **Named fields in variants** (`Rect { w: u32, h: u32 }`, Rust and Swift
  allow both forms). Not chosen now: it is a second form for the same thing,
  a variant holding a struct already names its fields, and `Shape.Rect { … }`
  in the condition of `if` or the subject of `match` would meet the rule of
  struct literals in heads ([GR-6]) again. Adding it later only accepts more
  programs.
- **Only named fields** (a variant always holds a struct, `Rect(Size)` with
  one type). Not chosen: `Some(v)`, the form everybody reads, is positional,
  and small variants (`Circle(u32)`) would need a struct each.
- **Unqualified construction and patterns** (`Circle(3)`, OCaml, Haskell). Not
  chosen: [DECL-6] already writes a variant with its enum, so that a name in
  a pattern is always a binding and a variant is never mistaken for one;
  `Circle(r)` in a pattern is refused with a fix to `Shape.Circle(r)`.
- **A variant with data written alone meaning any data** (`Shape.Circle` as
  a pattern, Swift's `case .circle:`). Not chosen: the rule of `Some(_)`
  applies, a value that is not tested is `_`, so the pattern says how many
  values the variant holds; the fix writes the `_`s.
- **Structural equality** (`==` on every enum whose data are comparable, field
  by field). Not chosen: which types are comparable (structs, and floating
  point, where `NaN != NaN`) is the open question of `OPEN.md` #32; a derived
  equality is better taken with traits. The cost of the refusal: code that
  compared an enum without data with `==` stops compiling when a variant gains
  data, and gets a note to use `match`.
- **Overlapped payloads** (a tagged union: the size of the largest payload
  instead of the sum; C, Rust, Swift, Zig). Not chosen now, and recorded in
  `OPEN.md` #35 as the next step. The size of the overlapped area would come
  from the compiler's own layout (`layout.rs`), not from LLVM's, while the
  build does not yet check that the target's data layout is the one the
  compiler assumes (F10 of the architecture audit): a mismatch would become a
  write out of bounds, not only a wrong fact. With every payload a member of
  its own, LLVM computes every offset and size, as for structs and `Result`.
  The overlap needs that check (audit §5.3.1 E, item E7) and the B0
  measurements, and changes no valid program, since the layout is internal.
- **Niches and field reordering** (the tag stored in unused values of a
  payload, Rust's `Option<&T>`; payloads reordered to reduce padding). Not
  chosen: the same reasons, and `OPEN.md` #35 keeps them open for
  measurement.
- **Changing `Option` and `Result` to the layout of user enums**. Not chosen:
  `Option` is already `{ tag, value }`, and `Result` already keeps both
  payloads, which is the layout chosen here; changing how they are lowered
  (first-class values unless they hold a struct) would change the code of
  every program for no gain.
- **Boxes for recursive enums** (a list as `Cons(u8, List)`). Needs heap
  allocation and ownership (`OPEN.md` #6); refused until then, with the
  diagnostic of an infinite size.
- **`..` in struct patterns** (Rust's `P { x: 1, .. }`). Not chosen: a field
  added to a struct would then be ignored by every pattern that uses `..`
  without saying so (M-2); with every field written, the new field is an
  error at each pattern, as it is at each literal. `..` gets a mechanical
  fix, since it stands for exactly the fields it leaves out. Accepting it
  later only accepts more programs.
- **Field shorthand in patterns** (`P { x, y }` binding `x` and `y`). Not
  chosen: literals have no shorthand (`OPEN.md` #40), and a pattern should
  read like the literal it matches; decided together for both.
- **Missing fields as wildcards** (a pattern names only the fields it
  tests). Not chosen: the same silent tolerance as `..`, without even the
  `..` that shows it.
- **Positional struct patterns** (`Point(0, y)`). Not chosen: structs are
  built by name, and a pattern by position would depend on the order of the
  declaration.
- **Destructuring declarations** (`Point { x: a, y: b } := p`). Deferred: a
  declaration cannot fail, so it would need an irrefutable pattern rule; a
  `match` with one arm does it today. Recorded in `OPEN.md` #33.
- **The bound of the analysis.** No bound: the worst case is exponential, so
  a program of a few kilobytes could make the compiler run for hours
  (audit §5.3.1, B7). A limit in time: the same program would compile on a
  fast machine and fail on a slow one, and the result would not be
  deterministic. A syntactic limit (arms × fields): refuses large matches
  that are cheap to check (a `match` of 256 variants) and admits small ones
  that are not. Another algorithm (decision trees, backtracking automata):
  deciding exhaustiveness of patterns with several fields is coNP-complete
  (the arms of a `match` on a tuple of booleans are the terms of a formula in
  disjunctive normal form, which is exhaustive exactly when the formula is a
  tautology), so every exact algorithm is believed to have an exponential
  worst case; a better algorithm moves the bound, it does not remove it. Chosen: a budget of work of the algorithm,
  counted in a unit that depends only on the patterns and their types, so the
  answer is the same everywhere ([TGT-2]), with its own diagnostic.

## What could change it

- Generics (`OPEN.md` #5): generic enums, and `Option` and `Result` written as
  ordinary generic enums of a standard library.
- The check of the target's data layout in the build (audit F10, E7) and the
  B0 measurements: overlapped payloads, niches (`OPEN.md` #35).
- Traits or derived equality (`OPEN.md` #32): `==` on enums with data.
- Programs that need `..`, shorthand, or-patterns, guards or ranges often
  enough (`OPEN.md` #33, #40, #46); named fields in variants.
- Ownership (`OPEN.md` #6): moves instead of copies of enums with data, and
  boxes for recursive enums.
- The MIR-0 and its verifier (master plan, Phase 6): the lowering of `match`
  on a control-flow graph, which will check that no payload is read before its
  tag.
- The author's review, at the gate of 0.1, of the questions of the review of
  2026-09-28 (control repository): the budget of [MATCH-6] (its value and
  unit, and a minimum fixed by the specification, when generators of real
  programs exist); `..` in struct patterns, at least for the private fields
  of a struct of another package (`OPEN.md` #33); a test of a variant
  without `==` (`OPEN.md` #32); payloads summed or overlapped (`OPEN.md` #35).
