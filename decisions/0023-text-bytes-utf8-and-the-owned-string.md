# Decision 0023: Text — bytes, borrowed UTF-8 and the owned string

- Status: Accepted — **experimental**, under the provisional surface R-7
  (the decision register of the control repository,
  `plans/decisions/README.md`): ownership (C), the explicit copy S1,
  allocation (d) and out of memory O2. Reviewed by the author at the gate
  of 0.2, with decisions 0020 to 0022.
- Date: 2026-10-04
- Origin: OPEN #7 (bytes, code points and grapheme clusters); the author's
  requirements M-2 (nothing silent), M-3 (bytes, borrowed UTF-8 text and
  the owned string are distinct types; byte, code point and perceived
  character are not confused), M-4 (no abort hidden behind an API
  presented as recoverable) and P-2 (traceable costs), which answer Q1/A12
  with requirements on memory; task TASK-20260926-043 and its gates in the
  master plan (`plans/reviews/2026-09-26-master-plan.md`, "Portões
  objetivos (errata)", §4); the part of M-4 that decision 0022 moved here
  (the first operations that allocate).
- Spec: `semantics.md` §4.1 ([TEXT-1]–[TEXT-15]), §4.4 ([STR-1]–[STR-5]),
  [OWN-6], [DROP-4], §7.2 ([ALLOC-1]–[ALLOC-4]), [ABI-3] (a), §12;
  `modules.md` [STD-8]; `syntax.md` [LEX-18] (unchanged); `OPEN.md` #6,
  #7, #41.

## Context

Decision 0015 gave the language borrowed text: string literals of type
`&str`, valid UTF-8, `.len` in bytes, `.bytes`, byte equality, and every
other operation refused. Every `&str` viewed constant data, so it could be
returned freely ([TEXT-6]), and decision 0015 recorded the constraint this
puts on ownership: the rule for text that views other data must keep `fn
f(s: &str) -> &str { return s }` valid, or give such text another type.

Since then decision 0020 gave values a single owner, `inout` and views with
a lexical duration; decision 0021 gave structs destructors that run on
every exit; decision 0022 gave the standard library its layers, the policy
of allocation (d) and of running out of memory O2, and declared, without
implementing them, the owned string's place (`alloc/string`) and the shape
of its operations. What OPEN #7 still asked: text made from bytes, which
must be validated and must not trap; code points and grapheme clusters,
with the Unicode version written; parts of text at boundaries; the owned
string, its destruction, its copies and its allocations, which must appear
in the facts (P-2); and no API that says "character" without its unit.

## Decision

### 1. Three kinds of text value, never converted implicitly (M-3)

| Type | What it is | Where |
|---|---|---|
| `&[u8]` | bytes: a view of an array of `u8`, any values | [REF-1]–[REF-4] |
| `&str` | borrowed text: a read-only view of bytes that are valid UTF-8 | [TEXT-1]–[TEXT-15] |
| `string.String` | the owned string: UTF-8 bytes on the heap that it owns and frees | `alloc/string`, [STR-1]–[STR-5] |

No operation turns one into another without being written: `t.bytes` views
the bytes of text ([TEXT-4]); `utf8.from_bytes(b)` checks bytes and gives
text or an error ([TEXT-11]); `string.from(t)` copies text into a new owned
string and `string.as_str(s)` views the text of one. Text has three units,
and every name says which one it counts ([TEXT-15]): bytes (`.len`),
code points ([TEXT-13]), grapheme clusters ([TEXT-14], not in this version).

### 2. Text that views other data: one type, with sources ([TEXT-8], [TEXT-9])

A `&str` may now view the bytes of a place of the function: an owned
string, an array, or another text. It keeps one type: the alternative that
decision 0015 left open (a type of its own for such views) would split
every text API in two. Instead the compiler derives, from the expression
alone and with no lifetime written ((C) of R-7), the **sources** of every
value that holds text or is a slice: the locals of the function whose bytes
it may view.

- A string literal views nothing (constant data). A parameter views
  nothing the function keeps: its caller keeps what it lent ([OWN-3]).
- `&a`, an owned string given to `string.as_str`, or any place of a type
  that moves given to a parameter without `take`, views that local (for a
  binding of a `match` on a place, the subject's local, [OWN-5]; for a
  parameter without `take` or `inout`, nothing).
- `t.bytes`, `Some(t)`, `Ok(t)`, `Err(t)`, `t?` and a `match` (its arms'
  values; a binding, its subject) view what `t` views.
- An element of a view `&mut [T]` (or a part of one) views **that view**,
  also when it is a parameter. The view is the only access to its array
  while it lives ([REF-6]), so the array cannot change under the text but
  through the view: text that views an element through it freezes the
  view while the text lives (assigning an element through it, E0363;
  lending one `inout` or giving the view to a call, E0367), and text that
  views an array a live `&mut` view borrows is a use of it (E0366). A
  function may still return the text of an element of a `&mut [T]`
  parameter, which its caller keeps; returned through a view declared in
  the function, it views that view's array (E0750).
- **The result of a call that holds text views what its arguments view**,
  but those given to `take` parameters, which the callee owns. So a
  function may return text of a parameter it borrows, and `fn f(s: &str)
  -> &str { return s }` stays valid ([TEXT-6]).

A view lives as views of arrays do ([REF-7]): held by a variable, until the
end of the block that declares it; given to a call, until the call returns;
the subject of a `match`, while its arms run; the left operand of `==`,
until the right one is computed; the slice a `for` iterates, until the loop
ends. **While a view lives its sources do not change**: assigning one,
moving it, lending it `inout` or viewing it with `&mut` is an error (the
analysis of moves, E0363, E0366, E0367 of the compiler), so text never
outlives or observes a change of what it views. Two rules keep a view from
leaving its sources' scope:

- **E0750**: a function returns (also as the error of `require`, or the
  error `?` gives back) only text that views nothing of its own: constant
  data or its borrowed parameters. A view of a local, a `take` parameter
  or an `inout` parameter would outlive it.
- **E0751**: a variable holds only views of what its declaration viewed:
  `mut u := "x"` then `u = string.as_str(s)` is an error (declare a new
  variable), since `u` may outlive `s`.

Text, and an `Option` or `Result` that holds it, may be a parameter, a
local variable or a result; it is still never a field, the data of a
variant or an element of an array or slice ([TEXT-10]), so that no value
stores a view.

### 3. Bytes to text: validated, with a `Result` ([TEXT-11])

`utf8.from_bytes(b: &[u8]) -> Result<&str, utf8.Utf8Error>`, in the package
`core/utf8` (the `core` layer: no allocation), gives `Ok` with text that
views the same bytes, nothing copied, when they are UTF-8 as RFC 3629 and
Table 3-7 of Unicode define it, and otherwise `Err`: `Invalid(i)` when the
sequence that starts at byte `i` is not UTF-8 (a byte that starts none —
a continuation byte, `C0`, `C1`, `F5`–`FF` —, an overlong form, a surrogate
`U+D800`–`U+DFFF`, a value above `U+10FFFF`, a sequence cut by a byte that
does not continue it), `Truncated(i)` when the bytes end inside a sequence
valid so far. It never traps (M-2, M-4). No operation available to a
program makes text from bytes without this check: the operations the
library builds on (`text_of_bytes` and its siblings) are names in the
packages of the standard library only.

### 4. Boundaries and parts of text: `Result`, not a trap ([TEXT-12])

The gate let the decision choose a trap with a `T` code or a `Result` for a
part of text that does not start or end at a boundary. **`Result`**:
`utf8.slice(t, start, end) -> Result<&str, utf8.SliceError>` gives the part
from byte `start` to byte `end`, which views the same bytes, or
`Err(OutOfRange)` (`start > end` or `end > t.len`) or `Err(NotBoundary)`
(either inside the encoding of a code point); it never traps.
`utf8.is_boundary(t, i)` and `utf8.next_boundary(t, i)` say where code
points start, and `utf8.byte_at(t: &str, i: usize) -> Option<u8>` is the
byte at offset `i`, or `None` when `i` is not below `t.len`, with no trap
either ([TEXT-4]). The operator form `t[i..j]` stays refused, with the
sub-slices of OPEN #47: when it comes, it is the trapping form of the same
operation, and this one its recoverable form.

### 5. Code points: only through an explicit API ([TEXT-13])

A code point is a Unicode scalar value held in a `u32`; there is no `char`
type. `utf8.count_code_points(t)` counts them (the bytes that do not
continue a sequence), `utf8.code_point_at(t, i)` gives the one whose
encoding starts at byte `i`, or `None`, and `utf8.next_boundary(t, i)` steps
to the next. Nothing counts or iterates code points implicitly: `.len` and
`for b in t.bytes` are bytes.

### 6. Grapheme clusters: outside this version, with their Unicode version ([TEXT-14])

Grapheme clusters are **the extended grapheme clusters of UAX #29**, of the
**Unicode version the package states**, 16.0.0 for the first one. They need
no allocation nor operating system, so they belong to the `core` layer, in
a package of their own (`core/grapheme`), so that only a program that
imports them carries their tables. **Not in this version**: the tables of
`Grapheme_Cluster_Break`, `Extended_Pictographic` and `InCB` are a
deliverable of their own. The values derived by hand for the gate (below)
are that package's first conformance cases.

### 7. Units in names ([TEXT-15])

No operation of the specification or of the standard library is named
after "characters" or says "character" without its unit. Names say
`byte`, `code_point` or `grapheme`; `.len` is bytes, and so is every offset
of `core/utf8`.

### 8. The owned string ([STR-1]–[STR-5])

`String`, of the package `alloc/string` (the `alloc` layer), is owned UTF-8
text on the heap: a struct whose fields are private (the address of its
block, its length and its capacity) and whose **destructor frees its
bytes** ([DROP-1]). It is therefore a move type with a destructor
(decision 0021): it is destroyed exactly once, by its owner, on every exit
of the block that owns it — the end of a body, `return` (also in an arm), a
false `require`, a `?` that returns, `break`, `continue` —, never after a
move, and never by a trap ([DROP-3], [TRAP-1]). A `String` is made only
from text, so it always holds UTF-8.

| Operation | Plain form (traps T0007) | `try_` form (`Result<…, alloc.AllocError>`) |
|---|---|---|
| empty, no allocation | `string.new() -> String` | — |
| empty, room for `n` bytes | `string.with_capacity(n)` | `string.try_with_capacity(n)` |
| from text, room for exactly it | `string.from(t: &str)` | `string.try_from(t)` |
| append text | `string.push(inout s, t: &str)` | `string.try_push(inout s, t)` |
| copy, deep | `string.copy(s)`, also written `s.copy()` | `string.try_copy(s)` |
| length in bytes, capacity | `string.len(s)`, `string.capacity(s)` | — |
| its text, a view | `string.as_str(s) -> &str` | — |

- **Growth** (`push`): when the block is full the new one is twice as
  large, at least 8 bytes and at least the new length. It is part of the
  documented behaviour (`capacity`), so that the cost of a series of pushes
  is predictable.
- **Out of memory, O2** ([STR-3], [ALLOC-3]): the plain form traps with
  *allocation failed* (`T0007`, reserved by decision 0022 and now
  reported), at the operation in `alloc/string`, and its documentation says
  so; the `try_` form returns `Err(CapacityOverflow)` when the size does
  not fit (above the largest `isize` of the target, or a length that
  overflows) and `Err(OutOfMemory)` when the allocator has none, never
  traps, and leaves its `inout` string as it was. `try_copy` is the
  recoverable copy that decision 0022 left out of O2 as additive.
- **Copies** ([STR-4], [OWN-6], [DROP-4]): `s.copy()` of a `String` is
  **deep**: it calls `string.copy`, which allocates its own block (it traps
  like a plain form). The copy and the original own different bytes, so
  both are destroyed. A value that **holds** a `String` (a struct, an enum,
  an `Option`, a `Result`, an array) is still never copied, and `[s; N]` of
  a `String` is refused: a deep copy of an aggregate would need copy
  functions for every type, which waits for traits (OPEN #5). (Decision
  0022, part 4, had `[v; N]` of an owned heap value make N − 1 copies that
  allocate; this decision refuses it, as [DROP-4] refuses it for every
  value with a destructor, and [ALLOC-3] says so.)
- **Nothing implicit** ([STR-5], the gate's condition on Q1/A12): no
  operator and no conversion allocates or copies text; `+` on text is still
  refused (E0320). Every allocation is a call of a function of
  `alloc/string`, written in the program, and the facts of `inspect` say
  which functions allocate (`allocates`, of every function and every call,
  through what it calls) and where the library asks for memory
  (`allocations`). The deep copy is a call too (`calls`), and every
  destruction is in `destructions` (decision 0021).
- **The allocator** of the hosted runtime is the C library's `malloc`,
  `realloc` and `free`, which become symbols of the hosted runtime that no
  `extern` or `export` function may take ([ABI-3] (a)): a program that
  defines them would change what the code of `effects {}` functions does.
  Bytes are copied into the block by the runtime's own routine, which calls
  no C function. A freestanding program has no allocator yet (OPEN #41).

### 9. Literals ([LEX-18], gate item 3)

The lexical rule for string literals is decision 0015's [LEX-18], which
replaced the restriction of [LEX-11] and [LEX-17] (only the path of an
import was quoted): printable ASCII and escapes, `\u{…}` for any scalar
value; [LEX-3a] holds inside literals, and a bidirectional control in a
literal is refused with the code of one in a comment (the compiler's
`fail/bidi_in_string_literal` and `fail/bidi_in_comment`, E0105). This
decision changes no lexical rule.

### 10. Values derived by hand (the gate's table)

| Text | Bytes | Code points | Grapheme clusters (UAX #29, not offered yet) |
|---|---|---|---|
| `ação` (`a\u{E7}\u{E3}o`) | 6 | 4 | 4 |
| `e` + U+0301 (`e\u{301}`) | 3 | 2 | 1 |
| U+1F600 (`\u{1F600}`) | 4 | 1 | 1 |
| U+1F1E7 U+1F1F7, the flag of Brazil | 8 | 2 | 1 |

Invalid sequences, each `Err` of `from_bytes` at offset 0: `C0 80`
(overlong, `Invalid`), `ED A0 80` (a surrogate, `Invalid`), `E2 82` (cut
short, `Truncated`), `F4 90 80 80` (above U+10FFFF, `Invalid`). The
compiler's tests run these in the three engines on both macOS targets and
in the reference evaluator on `i686-unknown-linux-gnu`.

### 11. Closes OPEN #7

OPEN #7 points here. What stays open is listed there: grapheme clusters
(above), ordering and collation, formatting into text, UTF-8 written as
itself in literals (with OPEN #10), byte-string literals, `t[i..j]` (with
OPEN #47), and deep copies of values that hold a `String`.

## Alternatives

- **Validation that traps**, or text made from bytes without a check in
  safe code: the first hides an abort where bytes come from outside (M-2,
  M-4), the second breaks [TEXT-2]. Not chosen.
- **A trap for a part not at a boundary** (with a code of its own) and the
  syntax `t[i..j]`: shorter, but a trap is the wrong default for offsets
  that come from data, and the sub-slice syntax is OPEN #47's. Chosen: the
  `Result` form now; the trapping operator can come with #47.
- **A type of its own for views of non-constant text** (decision 0015,
  item 6): sound and simple to check, but every text function would exist
  twice, or need generics. **Lifetimes written** (the Rust family, (A) of
  the study of TASK-040): refused by R-7. **Never returning text**: would
  invalidate valid programs (OPEN #48). Chosen: one `&str`, with sources
  derived from the expression and a lexical duration, which needs nothing
  written and refuses only views that would outlive their sources.
- **`from_bytes` copying into a `String`**: no view of the bytes to track,
  but it allocates for every validation and puts validation in `alloc`;
  the view keeps it in `core`. A copying form is additive
  (`string.from(utf8.from_bytes(b)?)` writes it today).
- **The owned string as a type built into the compiler**: every part of
  the compiler would learn a new type. Chosen: a struct of the library with
  a destructor (decision 0021), built on a few operations of the library's
  own (the heap, text from checked bytes), which only the packages of the
  standard library can name, and which the three engines carry out.
- **`s.len` on the owned string**: its fields are private; `string.len(s)`
  says the unit like `.len` of text does.
- **A small-string optimization** or another growth factor: not chosen; the
  growth above is simple, deterministic and documented.
- **A `char` type** for code points: a code point is a `u32` the API names;
  a type may come with formatting and pattern matching on code points.
- **Grapheme clusters now**: a partial segmentation (without the full
  tables) would be wrong for real text; the package comes with its tables.
- **`malloc` reached through the program** (an `extern fn`): the allocator
  must be the runtime's, the same for every `effects {}` function, so its
  symbols are reserved like `write` and `getenv`.

## What could change it

- The author's review at the gate of 0.2, with decisions 0020 to 0022.
- Generics and traits (OPEN #5): copy functions for aggregates that hold a
  `String`, `List<T>` and `Box<T>` of `alloc`, an iterator of code points.
- OPEN #47 (sub-slices): `t[i..j]` as the trapping form of `utf8.slice`.
- The freestanding profiles (OPEN #41): an allocator a program supplies.
- The text benchmarks (B1 of the compiler) and real programs: the growth
  factor, a byte search in the library, a copying `from_bytes`.
- Grapheme clusters (`core/grapheme`), with the tables of a stated Unicode
  version; formatting into text; ordering.
