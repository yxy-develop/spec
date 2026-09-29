# Decision 0012: Structs as value types, and their layout per target

- Status: Experimental — plans/decisions/README.md, L-0012
  (proposed under task TASK-20260925-002; open to the author's review)
- Date: 2026-09-25
- Spec: `syntax.md` [LEX-5], [LEX-6], [NL-6], [NL-8], [GR-6], [GR-7];
  `semantics.md` §2.1 [STRUCT-1]–[STRUCT-9], [TY-1], [DECL-2], [ORD-5],
  [ORD-6], [NUM-5], [ABI-2], [TGT-1]

## Context

Most real programs group values: a position, a range, a request with its
limits. Until now `struct` was a reserved word rejected as "not supported yet"
(open decision 5). Structs are the first user-defined aggregate, so they fix
several things at once: how aggregates are written and read, how they are
copied, in which order their parts are evaluated, how `{` after a name is
parsed, and what their memory layout is. Yxy is multi-architecture by design
(decision 0010): the layout must depend on the target, never on the machine
the compiler runs on — for example a `{ u8, u64 }` is 16 bytes with 8-byte
alignment on x86-64 and AArch64, but 12 bytes with 4-byte alignment on 32-bit
x86 Linux, where the System V i386 ABI aligns 64-bit integers to 4 bytes.

## Decision

1. **Declaration and literals.** `struct Name { field: Type, ... }` with
   fields separated by `,` or new lines (like enum variants). A literal
   `Name { field: value, ... }` gives every field exactly once, in any order;
   missing, unknown and repeated fields are errors; there are no defaults,
   no shorthand and no update syntax. Field types are the value types except
   arrays; slices and references are not fields in this version; a struct
   that contains itself (directly, inside `Option`/`Result`, or through other
   structs) is an error; a struct has at least one field and, in this
   version, at most 16 384 scalar components and 256 levels of nesting (see
   below).
2. **Value semantics.** Structs are copied on declaration, assignment,
   passing, returning and storing. There is no aliasing. Fields are read from
   any struct value (`make().x`, `a[i].x`, `s.a.b`) and assigned only through
   a `mut` local variable (`v.f.g = value`); parameters stay immutable.
3. **Nothing implicit.** No equality or ordering operators, no methods, no
   generics, no patterns for structs (a `match` on a struct is an error), no C
   ABI. A struct value is never discarded silently. *(Patterns: changed by
   decision 0017, which gives structs struct patterns, `S { f: p, … }`, and
   makes a `match` on a struct valid.)*
4. **Evaluation order.** Field values are evaluated in the order they are
   **written**, each completely, before the struct exists; a failure or a `?`
   in one field prevents the later ones. `v.f = value` evaluates `value`, then
   stores, so `value` may read `v`.
5. **`{` after a name in a condition.** In the condition of `if`/`while` and
   the subject of `match` (outside nested delimiters), `Name {` starts the
   block; a struct literal there is written in parentheses. The compiler
   recognizes an unparenthesized literal that is followed by an operator, `.`,
   `?`, `[` or `{` and reports exactly that, with a mechanical fix that adds
   the parentheses; the formatter always writes them.
6. **Layout: internal and target-defined.** Fields in declaration order with
   natural padding; the struct aligned to its most aligned field and padded to
   a multiple of it; `bool` and enums 1 byte, `()` 0 bytes, integers their
   width with the alignment of the target's data layout; `Option<T>` as
   `{ u8, T }`, `Result<T, E>` as `{ u8, T, E }`. The compiler records the
   integer alignment of each target in its target table (taken from, and
   tested against, the LLVM data layout that clang uses for the triple), and
   tools report sizes, alignments and offsets (`yxy inspect --json`). The
   layout is **not** a C ABI promise and may change between compiler
   versions; it is compared with clang's layout of equivalent C structs only
   as independent evidence that the computation is right.
7. **Limits for this version.** Because a struct can contain two copies of
   another, a handful of declarations can describe a value with millions of
   components, and a chain of declarations can nest structs arbitrarily deep.
   A struct, and any `Option`/`Result` type, has at most 16 384 scalar
   components and 256 levels of nesting. The code generator keeps struct
   values (and `Option`/`Result` values holding a struct) in memory and copies
   them with `memcpy`, reading and writing a field of a place alone; on the
   development host the worst measured programs at the limit build in
   1.5–3.7 s at `-O2` (before that lowering, 1 024 fields took 14.5 s and an
   `Option` of a 4 096-field struct 71 s and 15 GB). The measures count
   components and levels, not bytes, so the same program is valid on every
   target.

## Alternatives

- **Declaration-order evaluation of literals** (C++ member initializers run
  in declaration order whatever the written order). Rejected: the order a
  reader sees would not be the order effects happen; Rust and Go evaluate in
  written order.
- **Positional literals** (`Point(1, 2)`, tuple structs) or **default field
  values**. Deferred: named, complete literals are unambiguous for readers and
  agents, and a missing field is always an error rather than a silent default.
- **Allowing unparenthesized literals in conditions** by deciding from names:
  `Name {` would be a literal when `Name` is a struct type. Rejected: the
  parser (and the formatter, and tools that read code without checking it)
  would need name resolution; the same text would parse differently depending
  on declarations elsewhere. Go and Rust both require the parentheses.
- **A different literal syntax** that never collides with blocks (Zig's
  `.{ ... }`, or `Point.{ ... }`). Rejected for now: `Name { field: value }`
  is the form most programmers and models already read without effort; the
  collision is limited to condition heads and diagnosed precisely.
- **Field reordering to minimize padding** (Rust's default representation).
  Rejected for now: declaration order is predictable for readers, tools and a
  future C-compatible mode; reordering can still come later because the layout
  is declared internal.
- **A C-compatible layout promise** (`repr(C)` by default). Rejected: it would
  freeze `Option`/`Result` representations and forbid future optimizations
  (for example niche tags); interoperation needs an explicit opt-in design.
- **Equality by fields** (`==` derived automatically). Deferred: equality of
  aggregates needs a decision on which types are comparable (and later, on
  floating point), better taken with traits or an explicit derivation.
- **Host-dependent layout.** Rejected by decision 0010.
- **A limit in bytes** (for example the largest object of a 32-bit target):
  the limit would depend on the target's alignments, and a struct of `()`
  fields has no bytes but still costs the code generator per component.
- **No limit.** Rejected: nesting makes the number of components exponential
  in the size of the source, and a chain of 20 000 nested declarations crashed
  clang (1 000 000 overflowed the compiler's own stack) before the depth limit.

## What could change it

- A C interoperation design (an explicit C-layout annotation and passing
  structs across the boundary by value or by pointer).
- Ownership and moves (milestone 0.2): measured cost of copying large structs
  may favour moves or borrows of structs.
- Generics and traits: generic structs, derived equality, methods.
- Patterns: destructuring structs in `match` and declarations.
- Layout optimizations (field reordering, niche tags for `Option`), allowed
  because the layout is internal.
- Measurements on real programs, or a code generator that no longer walks
  every component of a copied value, which would allow raising the limits of
  item 7.
- Evidence that the parenthesized condition rule confuses readers or agents.
