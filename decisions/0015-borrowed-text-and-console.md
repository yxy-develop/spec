# Decision 0015: Borrowed text and the console capability

- Status: Experimental — plans/decisions/README.md, L-0015
  (proposed with the task that anticipates a minimal core of text and
  printing, TASK-20260928-080, after the author's decision R-3 of 2026-09-28;
  open to the author's review)
- Date: 2026-09-28
- Number: 0014 is held by the register's L-0014 ([GR-3], recorded in
  `syntax.md` without a file of its own), so this decision takes 0015
- Amendment (2026-10-04, experimental): decision 0026 makes `Console` the
  first of five root capabilities, all given to `main` ([MAIN-1], [CAP-2]),
  and adds the console in memory, derived from a console, which writes to
  a stream in memory for tests ([CON-5]); item 8 reads "the authority to
  write to a stream: standard output, or a stream in memory"; the measure of
  the redundancy is decided there (part 10: both stay).
- Spec: `syntax.md` [LEX-11], [LEX-17], [LEX-18], [GR-1], [GR-7];
  `semantics.md` §2 (table), [PRG-2], §4.1 ([TEXT-1]–[TEXT-7]), [EFF-2],
  §7.1 ([CON-1]–[CON-4]), [TRAP-1], [MAIN-1], §12; `OPEN.md` #7 (its borrowed
  part) and #9 (its first capability)
- Author requirements it follows (register of the control repository):
  M-2 (nothing silent), M-3 (bytes, borrowed UTF-8 text and the owned string
  are distinct types; byte, code point and perceived character are not
  confused), M-4 (no abort hidden in an API presented as recoverable), E-1
  (a small, extensible effect system), E-2 (effects checked transitively),
  E-3 (capability ≠ effect; the console arrives through an explicit API,
  without global singletons), P-2 (traceable costs)

## Context

Before this decision the only quoted literal was the path of an import
([LEX-17]); `str`, `String` and `char` were refused; the only tracked effect
was `ffi`; `main` took no parameters, and how capabilities reach it was
"future work" ([MAIN-1]). Printing "hello" needed `extern fn write` and the
`ffi` effect, bytes spelled by hand, and ambient authority: any package could
declare `write` (the architecture audit of 2026-09-26, §12b, calls it the
contagion of `ffi`).

The author ordered (R-3) a minimal core of **borrowed** text and of printing
through the console, so that a preview is usable earlier. The owned string
waits for ownership (TASK-042), and the `core`/`alloc`/`std` layers for the
standard library (Phase 7). The master plan asks, for text, rules with ids
where `len` counts bytes, borrowed text that is always valid UTF-8, a lexical
rule for literals under which [LEX-3a] still holds, and no API that says
"character" without its unit ("Portões objetivos", §4); and, for the console,
the first capability, on direction (c) of the audit (§12b.1, C1: the effect
is the kind of action, checked by [EFF-3]; the capability is the value born
in `main`), with the redundancy of `out: Console` and `effects { console }`
measured before the catalogue of TASK-048.

## Decision

### Text

1. **String literals** ([LEX-18]). `"…"` closes on the line where it starts.
   Inside it only printable ASCII (U+0020–U+007E) stands for itself, `"` and
   `\` excepted; everything else is an escape: `\n`, `\t`, `\r`, `\0`, `\\`,
   `\"` and `\u{H…}`, with 1 to 6 hexadecimal digits naming a Unicode scalar
   value (not a surrogate, at most U+10FFFF). A non-ASCII character or a tab
   written as itself is an error whose mechanical fix is its escape
   (`ç` → `\u{E7}`), which gives the same bytes. [LEX-3a] holds inside
   literals as everywhere. The value of a literal is the UTF-8 encoding of
   its characters. The path of an import keeps its own rule ([LEX-17]).
2. **`&str`** ([TEXT-1], [TEXT-2]). A string literal has type `&str`: a
   read-only view of UTF-8 bytes that live for the whole execution (constant
   data of the program). Every `&str` is valid UTF-8: literals are checked
   when read, and no operation of this version makes text from other bytes.
   No text is copied or allocated. `&str`, `&[u8]` and the owned string are
   distinct types (M-3), with no implicit conversion.
3. **Length in bytes** ([TEXT-3]). `s.len` is the number of bytes of `s`, a
   `usize`: `"a\u{E7}\u{E3}o".len` (`ação`) is 6. No operation of this version
   counts code points or grapheme clusters, and no text API is named after
   characters without its unit.
4. **The byte view** ([TEXT-4]). `s.bytes` is a `&[u8]` over the same bytes,
   nothing copied, under the rules of slices ([REF-2]–[REF-4]).
5. **Equality** ([TEXT-5]). `==` and `!=` compare text byte by byte: equal
   when both have the same length and the same bytes. There is no
   normalization: `"e\u{301}"` (3 bytes) is not `"\u{E9}"` (2 bytes). There
   is no ordering.
6. **Where text goes** ([TEXT-6]). Parameters, local variables (also `mut`,
   reassigned with other text), function results and `match` values. Not in
   arrays, `Option`, `Result` or struct fields, and not at the C boundary.
   A function may return `&str` because every `&str` of this version views
   constant data. This differs from slices, which are never returned
   ([REF-2]), and it constrains ownership (OPEN #6): `fn f(s: &str) -> &str
   { return s }` is valid today, so the rule for text that views other data
   (the owned string, validated bytes) must either allow returning a borrow
   derived from a parameter, or keep `&str` for constant text only and give
   views of other data a type of their own. A rule that forbids returning
   any `&str` would invalidate valid programs (OPEN #48).
7. **Everything else is refused, each with a diagnostic of its own**
   ([TEXT-7]): concatenation, indexing, ordering, the owned string, `str`
   without `&`, `&mut str`, `char` and character literals, formatting
   (formatting functions, format strings, `print` with several arguments),
   methods and fields other than `.len` and `.bytes`, and `match` on text.

### The console

8. **`Console` is a capability** ([CON-1]): a value that carries the
   authority to write to the process's standard output. Its type name is in
   the prelude ([PRG-2]), and so is `str` (see "Names now reserved" below).
   No expression creates one: the runtime gives it to
   `main`, which may declare one parameter of type `Console` ([MAIN-1]),
   and a function that prints receives it as a parameter. There is no global
   console: there are no global variables ([INIT-1]), and `static` stays
   refused. A `Console` may be a parameter or a local variable, and copying
   it copies the authority, not the stream; it is not returned, stored in a
   struct, an array, `Option` or `Result`, and does not cross the C boundary,
   so that the parameters of a function show every console it can reach.
9. **Operations** ([CON-2]): `console.print(text: &str)`,
   `console.print_u64(value: u64)` and `console.print_i64(value: i64)`, all
   returning `()`. `print` writes the bytes of the text and nothing else (no
   line break); the integer operations write the value in decimal, `-` before
   a negative one, with no padding. The call is written with the capability
   before `.` ([GR-1]); it is the only call written with a value there, and
   `Console` has no other operation.
10. **Effect** ([CON-3], [EFF-2]). Each operation performs the tracked effect
    `console`: the function that calls it declares `console`, and [EFF-3]
    carries the effect to every caller. The capability and the effect are two
    things (E-3): the effect says what kind of action may happen and is
    checked transitively through signatures (E-2), so that `effects {}`
    keeps meaning "no tracked effect" and tools keep it as the upper bound;
    the capability says which stream. A function that holds a `Console`
    without declaring `console` cannot write, and one that declares `console`
    without receiving a `Console` cannot either.
11. **Writes and their failure** ([CON-4], M-2, M-4). An operation writes
    every byte before it returns, in order and unbuffered, so that what was
    written before a trap is kept ([TRAP-1]); empty text writes nothing. When
    the stream refuses a write — closed for writing, a device error, a full
    disk, a write interrupted by a signal — the program traps with the kind
    "console write failed" (`T0005` in the compiler), reported at the
    operation: one report on standard error and exit status 101. The
    operations are not presented as recoverable, so no abort is hidden in a
    recoverable API. On a pipe whose reader has gone, the operating system's
    default for SIGPIPE applies (it ends the process): Yxy does not change
    signal dispositions. *Known limit:* the runtime does not read `errno`,
    so a write interrupted by a signal (`EINTR`) before any byte is written
    is a failure (`T0005`), not retried; programs of this version install
    no signal handler, but foreign code may install one without
    `SA_RESTART`. Reading `errno` (a function of each C library:
    `__error` on Darwin, `__errno_location` on glibc) and retrying belong to
    the catalogue of TASK-048, with the failures other resources report.
12. **Runtime.** The hosted runtime writes with `write` alone, already a
    reserved symbol ([ABI-3] (a)); no C library function and no stdio
    buffer is used. A freestanding program has no console ([STD-2]).

### Names now reserved

`str` and `Console` join the prelude names ([PRG-2]), which nothing may
redefine. A program that used one of them as the name of an item, a
parameter, a local variable or an import (a `struct Console`, a local
`str`, a package named `str`) was valid before this decision and is refused
now (the compiler's E0205, with a note that names this decision; E0813 for
an import). No mechanical fix is offered: a rename changes every use, which
may be in other files of the package or in other packages. This is the cost
the alternative "a `print` function in the prelude" (below) avoids for
`print`; it is accepted for these two names because the type of the
capability must be one that no program can redefine, or `fn main(console:
Console)` would depend on the program, and because `str` names the type of
`&str` as the integer names name theirs. It is recorded in OPEN #48 as a
change that invalidates valid programs. `String`, `string` and `char` stay
ordinary names: a type of the program with one of them is that type, and
the refusal of [TEXT-7] applies only when the program declares none.

### Measure of the redundancy (for TASK-048)

In the programs of this change that print — the hello example, the text
programs of the compiler's `tests/run`, its golden formatter file and the
program of its facts test — 8 functions write to the console, and every one
has both a `Console` parameter and `effects { console }`; no valid program has
one without the other (only the rejection tests do, to show E0600 and
E0610). In this version the pair therefore carries one bit: a function can
write if and only if it has both. What the effect adds that the parameter
does not: a bound read from the signature alone that stays transitive when
capabilities can be stored in structs or captured (future), and the rule that
already checks `ffi`, so tools keep `effects {}` as the upper bound (audit
§12b.1, task 5 (e)). Its cost is one word in the signature of every printing
function. This decision keeps both (direction (c)); the catalogue of
TASK-048 decides with this measure.

## Alternatives

- **`.len` named `byte_len`.** It says the unit. Not chosen: `&str` is a view
  of bytes like `&[u8]`, whose `.len` counts its elements; the master plan
  asks that `len` count bytes; and no other operation may be called `len`.
  Adding `byte_len` later is additive, while removing `.len` would need the
  evolution rule of OPEN #48.
- **UTF-8 written as itself in literals** (Rust, Go). Shorter for Portuguese
  text. Not now: a character may be written composed or decomposed (NFC,
  NFD), or with a look-alike, and the bytes a reviewer sees must be the bytes
  the program holds, as [LEX-3a] asks of code. Allowing it later accepts more
  programs, so it is additive; a policy for normalization and confusables
  (with OPEN #10) comes first.
- **`\x` escapes.** Only `\x00`–`\x7F` keep text valid UTF-8, and they would
  be a second spelling of those code points; bytes belong to byte strings,
  which are not in this version.
- **Text never returned**, as slices ([REF-2]). Simpler, but then no function
  could give a name to a value (an enum's name, a greeting); returning is
  sound while every text is constant, and stays sound under the ownership
  rule above.
- **Text in structs, `Option` or `Result` now.** It needs the lifetime rules
  of views of non-constant text; deferred with ownership.
- **Validated conversion `&[u8]` → text** (`Result`, never a trap). Deferred:
  the result would view the bytes of an argument, which is the rule that
  ownership settles, and its error needs a type for text errors; recorded
  for the rest of the text work (master plan, TASK-043 item 2).
- **Ordering of text by bytes.** Not an order people read (no collation);
  deferred.
- **A `print` function in the prelude**, called `print(console, "x")`.
  It reserves prelude names (`print` would break programs that define it)
  and makes the capability an ordinary argument. (The two names this
  decision does reserve, `str` and `Console`, have the same cost: "Names now
  reserved" above.) Chosen instead: the call
  form `console.print(x)`, only on a value of type `Console`. The grammar
  already reads it (the form of `pkg.f(x)`, [IMP-6]), and the checker tells a
  capability (a local) from an import, whose names never collide ([IMP-5]);
  no function name is reserved; the authority is visible at the call; other
  method calls stay refused. If methods come, these become the methods of
  `Console` with no change to programs.
- **`Console` in `std`**, imported. There is no `std` yet (Phase 7); the
  prelude name can move there later with a mechanical fix (OPEN #48).
- **An effect `io`** for all input and output. Coarser, and it would have to
  split when files and the network arrive; the criterion of OPEN #9 names an
  effect after a kind of action that the runtime mediates, and `console` is
  the audit's entry (§12b.1, row 11) and a candidate of [EFF-2].
- **Only the capability**, the effect deduced from parameters (audit §12b.1,
  C1 (b)). The parameters bound what a function reaches only while
  capabilities cannot be stored or captured; `effects {}` would stop being a
  transitive bound (E-2). **Only the effect** (C1 (a)): ambient authority,
  against E-3. **Capability and effect unified** (C1 (d)): needs closures and
  effect polymorphism.
- **A failed write as a `Result`** the program must handle. More precise for
  recoverable output, but every print would need `?` or `_ :=`, and `main`
  returns `()` or `u8`. A trap is not silent (M-2), and the operations are
  documented as trapping (M-4); an operation that returns the error can be
  added later without changing these.
- **Ignoring SIGPIPE** at start. It needs `signal` in the runtime and changes
  a process-wide disposition that foreign code and child processes inherit.
- **Buffered output.** Faster, but output written before a trap could be lost
  and a flush at exit would be needed; a buffered writer can be a library
  later.
- **Standard error and input.** Not in this version.

## Fix round (2026-09-28)

A review of the implementation found three defects, corrected without a
change to the rules above: the repeat form of an array (`[v; N]`) did not
apply the element rules of `[a, b]`, so `[console; 2]`, `["hi"; 3]` and a
slice `[s; 2]` were accepted ([CON-1], [TEXT-6], [REF-2]); types of the
program named `String`, `string` or `char` were refused as text types,
although these are not prelude names; and the compiler's trap event gained a
value in an enumeration that had not been declared open (recorded as an
amendment of the compiler's implementation decision 0010). This section and
"Names now reserved" also record the constraint on OPEN #6 (item 6) and the
limit on `EINTR` (item 11).

## What could change it

- Ownership (OPEN #6, TASK-042): the owned string, views of non-constant
  text and the rules for returning or storing them.
- The rest of the text work (OPEN #7, TASK-043): validation of bytes, code
  points and grapheme clusters with their Unicode version, slicing at
  boundaries, the text benchmarks.
- The catalogue of effects and capabilities (OPEN #9, TASK-048), with the
  measure above: standard error, input, a console that writes to memory for
  tests (audit §12b.1, task 5 (c)).
- The author's review of the experimental decisions (A2).
