# Yxy semantics — experimental, slice 1

Status: **experimental**. Normative for the first implemented subset. Syntax is
in `syntax.md`; decisions with alternatives and risks are in `decisions/`.
"Error" below means the compiler rejects the program with a diagnostic. The
compiler's diagnostic codes are listed in the compiler repository
(`docs/diagnostics.md`); they are tooling, not part of the language.

## 1. Programs

- **[PRG-1]** A program is a package and the packages it imports
  (`modules.md`). A package is the `.yxy` files of one directory of a module,
  or one standalone file; each file starts with `package <name>`.
- **[PRG-2]** Items are enums, structs, functions and *(experimental,
  decision 0018)* constants ([CONST-1]). Item names are unique in
  the package, across its files (`modules.md` [PKG-4]). The prelude names —
  `bool`, the integer type names, `str`, `Console` *(decision 0015)*, `f32`
  and `f64` *(decision 0019)*, `Option`, `Result`, `Some`, `None`, `Ok`,
  `Err`, the operations in §6.4, §6.5 and §6.6 — can not be redefined,
  neither as items, import names, parameters or local variables.
- **[PRG-3]** An executable program has `fn main()` (§10). A package without
  `main` can be compiled to an object and linked with other code. An
  executable package cannot be imported (`modules.md` [PKG-8]).

## 2. Types

| Type | Values | Where it may appear |
|---|---|---|
| `bool` | `true`, `false` | anywhere |
| `i8 i16 i32 i64 isize` | two's-complement integers of that width | anywhere |
| `u8 u16 u32 u64 usize` | unsigned integers of that width | anywhere |
| `f32 f64` | IEEE 754 binary32 and binary64 floats *(experimental, decision 0019; §6.6)* | anywhere |
| `()` | the single unit value `()` | anywhere; a function without `-> T` returns `()` |
| `E` (an `enum`) | one of its variants, written `E.Variant`, or `E.Variant(v, …)` with the values the variant holds (§2.2) | anywhere except the C boundary ([ABI-2]) |
| `S` (a `struct`) | a value for each of its fields (§2.1) | anywhere except the C boundary ([ABI-2]) |
| `Option<T>` | `None` or `Some(v)` | anywhere; `T` is a value type |
| `Result<T, E>` | `Ok(v)` or `Err(e)` | anywhere; `T`, `E` are value types |
| `[T; N]` | exactly `N` elements (`N` a constant expression, [CONST-5]) | local variables only, created from a literal (`[a, b]` or `[v; N]`) or as the explicit copy of another (`a.copy()`, decision 0020) |
| `&[T]` | a read-only view of an array's elements | parameters and local variables only |
| `&str` | text: a read-only view of valid UTF-8 bytes (§4.1) | parameters, local variables, results and `match` values ([TEXT-6]) |
| `Console` | the capability to write to standard output (§7.1) | parameters and local variables only ([CON-1]) |

- **[TY-1]** *Value types* are `bool`, the integers, the floats *(decision
  0019)*, `()`, enums (also those whose variants hold data, §2.2), structs,
  and `Option` and `Result` of value types. *(decision 0020)* A value of a
  copy type ([OWN-1]) is copied on assignment, when passed and when
  returned; a struct, an enum with data, and an `Option` or `Result` that
  holds one move ([OWN-2]) and are copied only by `.copy()` ([OWN-6]).
  *(Before decision 0020 every value type was copied.)* Arrays and slices
  have value-type elements.
- **[TY-2]** `usize` and `isize` have the pointer width of the target's data
  model: 32 or 64 bits in this version. They are distinct types from `u32`,
  `u64`, `i32` and `i64` on every target. All other integer types have the
  same width on every target.
- **[TY-3]** *(decision 0017)* A variant of an enum may hold data (§2.2); an
  enum has between 1 and 256 variants. `Option` and `Result` are known to the
  compiler; user-defined generics do not exist yet. *(Before decision 0017,
  variants carried no data.)*
- **[TY-4]** There are **no implicit conversions**, and **no default integer
  type**. An integer literal takes the integer type expected by its context
  (annotation, parameter, the other operand, the return type…). With no
  context, it is an error. *(decision 0016)* The bounds of a range in `for`
  have `usize` as their context when neither bound nor the loop variable
  gives them a type ([LOOP-3]). A literal that does not fit its type is an error;
  a `-` directly applied to a literal is part of the literal, so
  `x: i8 := -128` is valid. *(experimental, decision 0019)* A float literal
  ([LEX-19]) takes the float type expected by its context in the same way;
  there is no default float type either ([FLT-2]). An integer literal is
  never a float (`x: f64 := 1` is an error) and a float literal never an
  integer.
- **[TY-5]** `None`, `Ok(…)` and `Err(…)` need an expected type. `Some(v)` can
  take its type from `v`.
- **[TY-6]** Arrays are created from literals, and *(decision 0020, arrays
  R2)* as the explicit copy of a named array, `b := a.copy()` ([OWN-6]).
  A whole array is never moved nor copied implicitly: `b := a` is an error,
  so no buffer is ever copied silently; an array is read elsewhere through a
  borrow (`&a`). *(Before decision 0020 no whole array could be copied.)*
  A `match` does not produce arrays or slices.
- **[TY-7]** An enum or struct is identified by the import path of its
  package and its name (`modules.md` [VIS-7]): two packages that declare
  `enum Error` declare two distinct types. Another package's type is written
  qualified, `pkg.Error`.

### 2.1 Structs *(experimental)*

- **[STRUCT-1]** `struct Name { field: Type, ... }` declares a struct with at
  least one field. Field names are distinct. A field's type is a value type
  other than an array: an integer, `bool`, `()`, an enum, `Option`/`Result` of
  value types, or another struct. Arrays, slices and references cannot be
  fields in this version. Struct names are items ([PRG-2]); fields are not
  names in scope and may repeat names used elsewhere. Structs have no generic
  parameters in this version.
- **[STRUCT-2]** A struct holds its fields inline, so no struct may contain
  itself — directly, inside `Option` or `Result`, or through other structs;
  such a struct would have infinite size and is an error.
- **[STRUCT-3]** A struct literal `Name { f1: e1, f2: e2, ... }` gives **every**
  field exactly once, in any order. A missing field, a field the struct does
  not declare, and a field given twice are errors. There are no default
  values, no field shorthand (`Name { x }`) and no update syntax
  (`Name { x: 1, ..other }`). Each value is checked against its field's type,
  from which an integer literal takes its type ([TY-4]).
- **[STRUCT-4]** `e.f` reads field `f` of any expression `e` of struct type:
  a variable, a parameter, a call result, an array element, another field
  (`a.b.c`), a parenthesized literal. Reading a field the struct does not
  declare is an error. `Option` and `Result` have no fields; they are
  inspected with `match`.
- **[STRUCT-5]** `v.f = value` (and `v.f.g = value`, at any depth) assigns one
  field of a **`mut` local variable** `v`; the other fields keep their values.
  A field of an immutable variable or of a parameter cannot be assigned. In
  this version a field of an array element or of a temporary (`make().x`)
  cannot be assigned either; an array element is replaced as a whole
  (`a[i] = Point { ... }`).
- **[STRUCT-6]** *(changed by decision 0020, 2026-09-29; origin: the
  author's decision on Q3 of 2026-09-26, that a struct moves by default and
  is copied only explicitly, never by size, and the provisional ownership
  surface the author authorized on 2026-09-29, R-7)* Structs are values
  ([TY-1]) with a single owner: declaring, assigning, passing to a `take`
  parameter, returning, and storing in `Option`, `Result`, arrays or other
  structs **move** the value ([OWN-2]), and the place it came from cannot be
  used until it is given a new value ([OWN-4]); a parameter without `take`
  borrows its argument for the call ([OWN-3], [OWN-7]). A struct is copied
  only by `x.copy()` ([OWN-6]), never implicitly and never by its size.
  There is no aliasing: writing a field through one variable never changes
  another variable, a caller's argument or a copy. *(Until decision 0020,
  declaring, assigning, passing, returning and storing copied the whole
  value.)*
- **[STRUCT-7]** In this version structs have no equality or ordering
  operators (`a == b` is an error; compare fields), no methods and no C ABI
  ([ABI-2]). *(decision 0017)* A struct is matched with struct patterns
  ([MATCH-5]), or bound whole by a name; before decision 0017 structs had no
  patterns and a `match` on a struct was an error. A struct value is never
  discarded silently ([DECL-5]).
- **[STRUCT-8]** The **layout** of a struct — its size, alignment and field
  offsets — is defined by the compiler for the selected target ([TGT-1]),
  never by the machine the compiler runs on. It is an **internal** layout,
  not a C ABI and not stable across compiler versions: programs cannot observe
  it except through tools (`yxy inspect --json` reports it). In the current
  compiler, fields are laid out in declaration order, each at the next offset
  that is a multiple of its alignment, and the struct is aligned to its most
  aligned field and padded to a multiple of that alignment; `bool` and enums
  without data take one byte, `()` none, integers their width, aligned as the
  target's data layout says (64-bit integers are 4-byte aligned on 32-bit x86
  Linux and 8-byte aligned on the other supported targets; `usize`/`isize`
  follow [TY-2]); `Option<T>` is laid out as `{ tag: u8, value: T }` and
  `Result<T, E>` as `{ tag: u8, ok: T, err: E }`; an enum with data as
  [ENUM-5] says. See `decisions/0012-structs-and-layout.md` and
  `decisions/0017-enum-payloads-and-patterns.md`.
- **[STRUCT-9]** *(limits of this version)* A value of a struct type, of an
  enum with data (§2.2), or of an `Option` or `Result` type, has at most
  16 384 scalar components and at most 256 levels of nesting. Components:
  every integer, `bool`, enum without data and `()` counts one, and so does
  the tag of every `Option`, `Result` and enum with data, including those
  inside nested structs and enums; an enum with data counts its tag and the
  components of the data of every variant. Nesting: each struct, enum with
  data, `Option` and `Result` that holds another value inline adds one level
  (a struct of scalars is one level), consistent with [GR-5]. The limits
  apply to every struct and enum declaration and to every `Option`/`Result`
  type a program writes or builds (`Some(v)`); an array counts as its element
  type (an array is never a field, and a whole array is copied only by
  `.copy()`, [TY-6]). Without them, a few
  declarations could describe a value of millions of components (each struct
  holding two of the previous one) or a chain of structs deep enough to
  exhaust the tools. Both measures are independent of the target ([TGT-2]).

### 2.2 Enums with data *(experimental, decision 0017)*

- **[ENUM-1]** A variant of an enum is a name, or a name followed by the
  types of the values it holds, in parentheses: `Rect(u32, u32)`. A variant
  written with parentheses holds at least one value (`V()` is an error). The
  types are value types other than arrays, as for struct fields
  ([STRUCT-1]): integers, the floats *(decision 0019)*, `bool`, `()`, enums,
  structs, `Option` and `Result`; slices, text and capabilities are not. Named fields
  (`V { a: T }`) are not supported: a variant holds a struct for them. An
  enum mixes variants with and without data in any order. An enum that holds
  itself — directly, inside `Option` or `Result`, or through structs and
  other enums — would have infinite size and is an error ([STRUCT-2]). The
  data of the variants of a `pub enum` is part of its interface: its types
  are public ([VIS-3]).
- **[ENUM-2]** `E.V(e1, …, en)`, or `pkg.E.V(e1, …, en)` for an enum of an
  imported package, builds variant `V` with one value per type it declares,
  each checked against its type (an integer literal takes its type from it,
  [TY-4]). The values are evaluated in the order they are written, each
  completely, before the enum value exists; a trap, or a `?` that returns, in
  one value prevents the evaluation of the later ones ([ORD-5]). A variant
  with data is always written with its values (`E.V` alone is an error), and
  a variant without data without parentheses (`E.V()` is an error).
- **[ENUM-3]** An enum with data is a value ([TY-1]) that moves as a struct
  does ([STRUCT-6], decision 0020): declaring, assigning, passing to a
  `take` parameter, returning and storing it in other values move it whole,
  with its data ([OWN-2]); it is copied only by `.copy()` ([OWN-6]); there
  is no aliasing. *(Until decision 0020 these copied it whole.)*
- **[ENUM-4]** `==` and `!=` are not defined for an enum that has a variant
  with data; it is inspected with `match` ([NUM-5]). An enum without data
  keeps them.
- **[ENUM-5]** The layout of an enum is internal, like that of a struct
  ([STRUCT-8]), and tools report it (`yxy inspect --json`). An enum without
  data is its one-byte tag. In the current compiler an enum with data is laid
  out as `{ tag: u8, payload of each variant, in declaration order }`: the
  tag is the variant's number in declaration order, counted from 0; the
  payload of a variant with data is laid out as a struct of its values, in
  order ([STRUCT-8]); the payload of a variant without data is empty (size 0,
  alignment 1). Payloads are not overlapped (`decisions/OPEN.md` #35). Only
  the tag and the payload of the variant a value holds are written, and a
  `match` reads a payload only where the tag says that its variant is the one
  held.

## 3. Names, declarations and mutability

- **[DECL-1]** `name := value` declares an **immutable** local variable.
  `mut name := value` declares a mutable one. `name: T := value` and
  `mut name: T := value` add a type annotation. `_ := value` evaluates and
  discards a value; *(decision 0020)* `_ := p` of a place moves nothing
  ([OWN-2]). `let` and `var` do not exist.
- **[DECL-2]** `place = value` assigns. The place is a `mut` local variable, an
  element `a[i]` of a `mut` array, or a field `v.f` (at any depth) of a `mut`
  struct variable ([STRUCT-5]). Slices are read-only and cannot be reassigned.
- **[DECL-3]** Blocks (`{ … }` of `if`, `while`, `loop`, `for` and match
  arms) open scopes; the variable of a `for` is in the scope of its body
  ([LOOP-1]).
  **Shadowing is not allowed**: a name cannot be declared while another
  declaration with the same name is visible in the function, and a local or a
  parameter cannot take the name of an item or of an import of its file
  (`modules.md` [IMP-5]).
- **[DECL-4]** The regions of a cell form one sequential scope: a name declared
  in a region is visible in the following regions, from its declaration on.
- **[DECL-5]** Every value is used explicitly: an expression statement whose
  value is not `()` is an error; write `_ := expr` to discard.
- **[DECL-6]** In a pattern, a bare name binds the value. When the matched type
  is an enum that has a variant with that name, the pattern is an error (it
  would silently match everything); the variant is written `Enum.Variant`,
  and a variant with data `Enum.Variant(p, …)` ([MATCH-4]).

### 3.1 Constants *(experimental, decision 0018)*

- **[CONST-1]** `const NAME: T := value` declares a **constant**, an item of
  its package ([PRG-2]); `pub const` makes it visible to other packages,
  which name it `pkg.NAME` (`modules.md` [VIS-1]). The type is written and
  is an integer type or `bool`. A constant is declared at package level
  only: a `const` inside a function is an error. Its name stands for its
  value wherever it is read; it cannot be assigned, borrowed, indexed or
  called, is not a type, and is not a pattern (a name in a pattern binds,
  [DECL-6]; `pkg.N` in a pattern, or a constant as the name of a struct
  pattern, [MATCH-5], is an error); no other item, parameter, local or
  binding of its package takes its name ([DECL-3]). *(decision 0019)*
  Floats have no constants: `f32` and `f64` are not types of a constant.
- **[CONST-2]** The value is a **constant expression**: integer literals,
  `true`, `false`, names of constants (`N`, or `pkg.N` of an import), the
  operators of `syntax.md` §4, grouping parentheses, and `widen(x)` of a
  constant expression `x` ([CONV-1]: it never traps and keeps the value).
  Nothing else is: no other call (not `checked_convert` nor the operations
  of §6.4: there is no `const fn`), no variable, field, `.len`, variant,
  `match`, float literal, text or array. It is typed as in a function body
  ([TY-4], [CONV-1]), and its type is the constant's.
- **[CONST-3]** The compiler computes the value of every constant of the
  program, used or not, before the program runs, by the rules of §6 on the
  target's data model ([TY-2]): operands left to right, `&&` and `||`
  evaluating their right operand only when it is needed ([NUM-1]–[NUM-5]).
  Computing a constant has no effects and runs no code of any package
  (`modules.md` [INIT-2]). An operation that would trap ([NUM-1]) makes the
  program invalid: an error at that operation, which names the kind of
  failure.
- **[CONST-4]** *(limits)* A constant that reads itself, directly or through
  other constants, has no value: an error. The **steps** of a constant are
  the literals, operators, `widen` and names of its value, where a name of
  a constant also counts the steps of that constant — the size of the value
  with every constant written out in place of its name. A constant of more
  than 1 000 000 steps is an error, and so is a length or a count
  ([CONST-5]) of more, counted the same way: an expression that is too long
  to be a constant is too long to be a length. Both are decided from the
  names, before anything is computed, the same on every target, so
  checking a program always ends.
- **[CONST-5]** The length of an array type `[T; N]` and the count of
  `[value; N]` are constant expressions of type `usize` ([REF-3]), limited
  as in [CONST-4] and computed as in [CONST-3]; a literal is the simplest.
- **[CONST-6]** *([TGT-2] for constants)* The value of a constant may depend
  on the target only through the range and width of `usize` and `isize`,
  and so may its validity, as a literal's does: `const WORDS: usize :=
  65536 * 65536` is 2^32 on a 64-bit target and an error on a 32-bit one;
  `3 << 31` in `usize` drops a bit on 32 bits only ([NUM-3]). A constant
  whose value reads no `usize` or `isize` value has the same value on every
  target; one that reads one may differ, whatever its own type (`WORDS > 0`
  of type `bool`, `widen(WORDS)` of type `u64`). Since every constant is
  computed, used or not ([CONST-3]), a `pub const` that is an error on
  32-bit targets only (`pub const BIG: usize := 1 << 40`) makes every
  program that imports its package invalid on those targets, even one that
  never reads it (`decisions/OPEN.md` #50).
- **[CONST-7]** *(certain traps)* An operation of a function body whose
  operands are constant expressions, and whose computation traps, does not
  make the program invalid: the program traps when it evaluates the
  operation ([NUM-1], [TRAP-1]). The compiler reports a **warning** there,
  one for each largest constant expression, at the operation whose trap the
  program would report; computing that expression reads the values of the
  constants it names, already limited by [CONST-4], and is not limited
  itself, since a warning decides nothing. An operation that reads a
  variable is not computed at compile time and gets no warning, even when
  it always traps (`x / 0`). An integer constant expression inside
  `to_float`, `fma` or the values of a variant ([FLT-6], [FLT-7], [ENUM-2])
  is one like any other; the operand of `truncate_to_int` ([FLT-8]) is a
  float, never a constant expression, so its trap gets no warning. A
  warning never makes a program invalid and never changes what it does; it
  may depend on the target ([TGT-2]).

## 4. Borrows

- **[REF-1]** `&a`, where `a` is an **immutable** array local, produces a slice
  `&[T]` over its elements. Borrowing a `mut` array, a scalar or a slice is an
  error. Because the borrowed array is immutable, a slice never observes a
  mutation. `for x in a` iterates `&a` and requires the same ([LOOP-4]).
- **[REF-2]** References cannot escape: no function returns a slice, and slices
  cannot be stored in arrays, `Option` or `Result`. A slice parameter or local
  therefore never outlives the array it views. This is a restriction of the
  subset, not a borrow checker; wider borrowing rules are future work. Text
  (`&str`, §4.1) views constant data, which outlives every function, so a
  function may return it ([TEXT-6]); the slice `t.bytes` of a text follows
  this rule like any slice.
- **[REF-3]** `s.len` is the number of **elements** of an array or slice, of
  type `usize`. In this version `s` names the array or slice: a local
  variable or a parameter.
- **[REF-4]** `a[i]` requires `i: usize` and checks `i < len` at run time (§6.1).
  In this version `a` names the array or slice, as in [REF-3]. The grammar
  accepts any operand before `[` and `.len`, but `(&a)[0]`, `[1, 2, 3][0]`
  and `(&a).len` are errors: write `a[0]` and `a.len`, or declare the value
  first, with its type when it is a literal (`d: [u8; 3] := [1, 2, 3]`,
  `s := &a`). The index is one expression: there are no sub-slices
  (`s[i..j]`, `decisions/OPEN.md` #47). This is a restriction of the subset,
  like [REF-2]: lifting it, with places that carry projections (OPEN #37,
  #47), changes no valid program.
- **[REF-5]** *(decision 0020)* Structs, enums with data, `Option` and
  `Result` that hold one, and arrays move ([OWN-1]); a use after a move is
  an error ([OWN-4]). *(Before decision 0020 every type was copied, and
  use-after-move could not occur.)*

### 4.1 Text *(experimental, decision 0015)*

Only borrowed text exists in this version; the owned string waits for
ownership (`decisions/OPEN.md` #6, #7). Bytes (`&[u8]`), text (`&str`) and
the owned string are distinct types, with no implicit conversion between
them.

- **[TEXT-1]** A string literal ([LEX-18]) has type `&str`: a read-only view
  of the UTF-8 encoding of its characters, which lives for the whole
  execution (constant data of the program). No text is copied or allocated
  in this version.
- **[TEXT-2]** Every `&str` value is valid UTF-8: literals are checked when
  they are read ([LEX-18]), and no operation of this version makes text from
  other bytes. A conversion of `&[u8]` to text, validated, returning a
  `Result` and never trapping, is future work (OPEN #7).
- **[TEXT-3]** `t.len` is the number of **bytes** of the text `t`, of type
  `usize`, for any expression `t` of type `&str`: `"a\u{E7}\u{E3}o".len`
  (`ação`) is 6. No operation of this version counts code points or grapheme
  clusters, and no text operation is named after characters without its
  unit.
- **[TEXT-4]** `t.bytes` is a `&[u8]` over the same bytes; nothing is copied.
  It follows the rules of slices ([REF-2]–[REF-4]): to index it, name it
  first (`b := t.bytes`, then `b[i]`).
- **[TEXT-5]** `a == b` and `a != b` compare two texts byte by byte: they are
  equal when they have the same length and the same bytes. There is no
  normalization: `"e\u{301}"` (3 bytes) is not equal to `"\u{E9}"` (2
  bytes). Text has no ordering.
- **[TEXT-6]** Text may be a parameter, a local variable (also `mut`,
  assigned other text), the result of a function and the value of a `match`.
  It cannot be stored in arrays, `Option`, `Result` or struct fields, and does
  not cross the C boundary ([ABI-2]). A function may return text because
  every `&str` of this version views constant data, also text it received
  as a parameter (unlike a slice, [REF-2]); when text that views other data
  exists, returning it follows the ownership rules, which must keep such
  programs valid (OPEN #6, decision 0015).
- **[TEXT-7]** Not in this version, each rejected with its own diagnostic:
  concatenation (`+`), indexing (`t[i]`), ordering (`<`), the owned string
  (`String`), `str` without `&`, `&mut str`, `char` and character literals,
  formatting (formatting functions, format strings, `print` with several
  arguments), sub-slices, fields and methods of text other than `.len` and
  `.bytes`, and `match` on text.

### 4.2 Ownership *(experimental, decision 0020)*

A value has a single owner. A value of a copy type is copied where it is
used; a value of a move type **moves**: the place it came from gives it
away. Only `x.copy()` copies a value of a move type. The direction of this
section is the ownership surface (C) of decision 0020 (values, `inout` and
limited borrows); this version has its part without `inout`, views with a
lifetime of their own or owned resources (TASK-20260926-042).

- **[OWN-1]** *(classes)* The **copy types** are `bool`, the integers, the
  floats, `()`, enums without data, `Option` and `Result` whose type
  arguments are all copy types, `&[T]`, `&str` and `Console` (copying a
  capability copies the authority, [CON-1]). The **move types** are every
  struct, every enum with a variant that holds data, `Option` and `Result`
  with a move type as an argument, and every array. No struct and no enum
  with data is a copy type; which types move depends on their declaration
  only ([OWN-8]).
- **[OWN-2]** *(moving uses)* A place ([REF-4], [STRUCT-4]: a variable, a
  field of a place, an element) of a move type **moves** when it is: the
  value of a declaration; the right side of an assignment (to a variable, a
  field or an element); an argument of a `take` parameter ([OWN-3]); the
  operand of `return` or the error of `require`; a field value of a struct
  literal, a value of a variant, the value of `Some`, `Ok` or `Err`, an
  element of an array literal; the value of `[v; N]`, which the repetition
  copies into the other elements ([ORD-4]: the one written form of copying
  besides `.copy()`); the value of a `match` arm; the operand of `?`, which
  moves the payload out of the place. Nothing else moves: `_ := p`,
  reading a field of a copy type (`p.x`), `.len`, comparisons, `&a`, an
  argument of a parameter without `take` ([OWN-7]), the subject of a
  `match` ([OWN-5]), the operand of `.copy()`. A value that is not a place
  (a call, a literal, a field of a temporary) is a new value, and using it
  moves nothing. A move is not written where it happens: the type and the
  position say it.
- **[OWN-3]** *(parameters)* A parameter `p: T` of a move type **borrows**
  its argument for the call: the function reads it, passes it to
  parameters without `take` and copies it, but never moves it or a part of
  it out ([OWN-5]); of a copy type, it is a copy, as before. A parameter
  `take p: T` **receives** the value: the argument moves ([OWN-2]), and the
  function owns it, as a variable declared with `:=`. `take` is written
  only before a parameter whose type moves, and is a word only there, so a
  name `take` stays usable. Parameters stay immutable bindings
  ([STRUCT-5]).
- **[OWN-4]** *(moved places)* After a moving use of a place `p`, `p` is
  moved on that path until a whole assignment `p = e` (a `mut` variable,
  [DECL-2]) gives it a value again. A use of `p`, or of a place that
  contains it or that it contains, where `p` is moved on every path that
  reaches the use, is an error; so is a use where `p` is moved on some of
  those paths only (**maybe-moved**: in one branch of an `if` or of a
  `match`, or in an earlier iteration of a loop that reaches the use
  again): nothing records at run time whether a value moved. Moving a field
  `p.f` of a variable that owns its value (a **partial move**) moves `p.f`
  and its parts; the other fields stay usable, and `p` as a whole is not,
  until `p` or `p.f` is assigned. Assigning a field of a moved place is an
  error.
- **[OWN-5]** *(borrowed places)* A value the function does not own is never
  moved out, whatever the position: a parameter without `take` and its
  parts ([OWN-3]); an element of an array or slice and its parts (`x :=
  a[i]`, `a[j] = a[i]`, `return s[0]` are errors: the array would be left
  with a hole); the variable of `for x in s` over elements of a move type,
  which borrows each element for one iteration ([LOOP-4]). A binding of a
  `match` that binds a part of a move type of a subject that is a place
  **is that part of the place**: the subject and the steps of the pattern
  to the binding, each a field of a struct or one value of a variant (of
  `Some`, `Ok`, `Err`, or a variant with data, whose values are parts
  apart). Moving the binding moves that part only, which is an error when
  the subject is a borrowed place (above, or such a binding), and otherwise
  makes a later use of the subject an error by [OWN-4]; the other bindings
  stay usable (`T.Two(a, b) => f(a, b)` with `take` parameters is valid),
  and a subject every arm of whose `match` moved a part is moved. A
  binding of a subject that is not a place owns its part. So a `match`
  consumes its subject only where nothing uses the subject after it. *(The
  review of TASK-20260926-078, 2026-09-29.)* **A binding borrows its
  part**: a binding is never a copy of its part ([OWN-8]), so while it
  lives (in its arm, until it moves) its arm neither assigns nor moves the
  subject, or a place that overlaps that part; the same holds for the place
  that holds the value of `x?` bound by a `match`. That is an error, whose
  fix is `match x.copy()`, a `match` on a copy the arm cannot change (or,
  for a move, `.copy()` where the place moves). A binding that moved
  borrows nothing, and its arm may give the subject a new value.
- **[OWN-6]** *(the explicit copy)* `x.copy()`, a postfix operation, gives a
  new value equal to the value of `x`, of the same type; `x` keeps its
  value, and the copy is independent of it (changing one never changes the
  other). It is not a moving use of `x`. It exists for exactly the move
  types of [OWN-1], all of whose parts copy in this version: structs, enums
  with data, `Option` and `Result` that hold one, and arrays (a whole array,
  copied element by element, [TY-6]); `.copy()` of a value of a copy type
  is an error. A copy costs the size of the value, which the tools report.
  A field named `copy` stays a field (`p.copy` reads it), and
  `pkg.copy()` is the function `copy` of an import ([IMP-5]: a variable and
  an import never share a name).
- **[OWN-7]** *(arguments are lent)* A place given to a parameter without
  `take` is **borrowed** from the moment it is evaluated until the call
  returns; it is not a copy taken when it is evaluated. A later argument of
  the same call that assigns or moves that place, or a place that overlaps
  it (one contains the other; two elements of one array, whatever their
  indices), is an error; a binding of a `match` lends with it the part of
  the subject it is ([OWN-5]). Writing `x.copy()` for the earlier argument
  passes the value as it was when it was evaluated; for a move, `.copy()`
  where the place moves keeps it unchanged. An argument of a `take`
  parameter moves when it is evaluated ([OWN-2]).
- **[OWN-8]** *(nothing silent, no size)* The compiler never inserts a copy,
  a reference count or any indirection to make a use valid: the fix its
  diagnostics suggest for [OWN-4], [OWN-5] and [OWN-7] is the explicit copy,
  `.copy()` where the value moves (after the earlier argument for
  [OWN-7], after the subject of the `match` for a binding, [OWN-5]), as a
  suggestion, never as a mechanical fix; a binding of a `match` is never a
  copy of its part. Whether a type
  moves depends only on its declaration, never on its size or on the target
  ([TGT-2]): a struct of one byte and one of 16 384 components move and
  copy alike. A move may be carried out by copying bytes; the tools report
  those copies with their size, and the explicit copies apart.

## 5. Functions and cells

- **[FN-1]** A function declares parameter types, its return type (or `()` when
  omitted) and its effects: `effects { … }` is mandatory (§7).
- **[FN-2]** A function whose return type is not `()` must return on every path:
  its body must not go on at its end ([LOOP-5]). Statements that can never run
  (after a `return`, a `break` or a `continue`, or regions after one that
  always returns) are errors.
- **[FN-3]** An expression that never produces a value (a `match` whose arms
  all return, or leave their iteration with `break` or `continue`, [LOOP-5])
  can only be used as a statement; binding it, passing it or returning it is
  an error.
- **[MATCH-1]** A `match` covers every value of the matched type — for a
  struct or a variant with data, every combination of the values of its
  fields ([MATCH-4], [MATCH-5]) —; otherwise it is an error that names one
  value not covered. Integer matches need a final `_` arm (an integer inside
  a struct or a variant needs `_` in some arm at its position).
- **[MATCH-2]** An arm that can never match, because earlier arms cover every
  value it matches, is an error.
- **[MATCH-3]** Without an expected type, the type of a `match` comes from its
  arms: an arm whose value needs a type from context (an integer literal,
  `None`) takes it from the other arms, whatever their order.
- **[MATCH-4]** *(decision 0017)* `E.V(p1, …, pn)` matches variant `V` of
  enum `E` when each value it holds matches the pattern in its position; a
  variant with data is matched with exactly one pattern per value (a value
  that is not tested is `_`), and a variant without data without
  parentheses. `Some`, `Ok` and `Err` take exactly one pattern. There are no
  rest patterns (`Some(..)`, `E.V(..)`).
- **[MATCH-5]** *(decision 0017)* `S { f1: p1, …, fn: pn }` matches a value
  of struct `S` when each field matches its pattern. Every field of `S` is
  written exactly once, in any order, as `field: pattern`; a field that is
  not tested is `field: _`. There is no `..` and no shorthand
  (`S { f }`); a missing, unknown or repeated field is an error. A struct of
  another package with a field that is not `pub` cannot be matched with a
  struct pattern ([VIS-2]). Struct patterns and patterns of variants nest in
  each other, and in `Some`, `Ok` and `Err`, at any depth.
- **[MATCH-6]** *(decision 0017)* Checking [MATCH-1] and [MATCH-2] for one
  `match` has a fixed budget of work, the same on every target and machine
  ([TGT-2]); a `match` whose check needs more is an error, with a note to
  split it, and the compiler never runs away on it. The budget, and the unit
  it is counted in, are the compiler's, documented with its diagnostics.
- **[CELL-1]** A **cell** is a function body written as regions. In this version
  a cell is the whole body. Regions appear in this order, each optional:

  | Region | Header | Meaning |
  |---|---|---|
  | control | `@ctrl:` | validations and conditions that govern the cell |
  | evaluation | `<- @eval:` | the central production; it may call operations with **declared** effects; it does not imply purity |
  | effect | `-> @effect:` (any number) | steps that make an action with effects explicit; effects may also happen elsewhere, and the label grants nothing |
  | output | `-> @out:` | delivery of the result or transfer of control, such as `return` |

  At most one `@ctrl`, `@eval` and `@out`. `@ctrl` takes no arrow; `@eval`
  takes `<-`; `@effect` and `@out` take `->`.
- **[CELL-2]** *(experimental; the place of `require` is the author's
  decision A1 of 2026-09-26)* Statements allowed in each region, at any depth
  (a statement inside an `if` or a loop follows the rules of its region):
  `@ctrl` — declarations and `require`; `@eval` — declarations, assignments,
  `if`, loops, calls and `require`; `@effect` — declarations, assignments,
  `if`, loops and calls; `@out` — declarations, `if`, calls and `return`.
  *(decision 0016)* Loops are `while`, `loop` and `for`, with `break` and
  `continue` inside them ([LOOP-6]).
  A `require` in `@ctrl` validates before the computation (pre-validation); a
  `require` in `@eval` validates what the statements of `@eval` before it
  computed (post-validation). It may appear anywhere in `@eval`: statements
  run in lexical order ([CELL-3]), so it sees exactly the values computed
  before it, and because `@eval` comes before every effect step and the
  output, a false `require` in `@eval` prevents them ([CELL-6]).
  Consequently validation (`require`) happens only in `@ctrl` and `@eval`,
  before every effect step, and an early exit before `@out` happens only
  through a false `require` or `?` (`break` and `continue` leave a loop,
  never a region). The rules hold within a cell, for that
  function: a function whose body is not a cell, or a function called from
  `@eval`, is not restricted by them. See `decisions/0004-cell-regions.md` for
  the alternatives and what would change them.
- **[CELL-3]** Execution follows the lexical order: control, evaluation, the
  effect steps, output; inside a region, statement order. The compiler does not
  run regions in parallel, speculate calls, move effects before validations or
  reorder operations that can fail.
- **[CELL-4]** `require cond else error` evaluates `cond`; when it is false it
  evaluates `error` and the function returns `Err(error)` immediately. It is a
  check performed at run time, never an assumption for the optimizer. The
  function must return `Result<T, E>` and `error` must have type `E`.
- **[CELL-5]** `e?` on `Result<T, E>` yields the `Ok` value, or returns the `Err`
  unchanged from the function, which must return `Result<_, E>` with the **same**
  `E`: errors are never converted implicitly. On `Option<T>` it yields the
  `Some` value or returns `None` from a function returning `Option<_>`.
- **[CELL-6]** Failures, by region. A false guard in `@ctrl` prevents the
  evaluation and every later step; the calls made by declarations of `@ctrl`
  before the guard have already run, with their effects. A failure in `@eval`
  — a `?` or a false `require` (post-validation, [CELL-2]) — prevents the rest
  of `@eval`, the effect steps and the output. A failure in an effect step
  does **not** undo the actions already performed and prevents the later
  steps. There is no implicit rollback, retry or exactly-once delivery;
  transactions and idempotence belong to explicit library contracts.
- **[CELL-7]** `return` leaves the function. `when` (a cell that is skipped when
  its condition is false) is reserved and not supported yet.
- **[CELL-8]** No value in this version owns a resource, so leaving a function
  runs no cleanup code. Deterministic cleanup order is future work.

### 5.1 Loops *(experimental, decision 0016)*

- **[LOOP-1]** `while cond { … }`, `loop { … }` and `for x in source { … }`
  are statements. `while` runs its body while `cond` is `true`, evaluating
  `cond` before each iteration; `loop` runs its body until a `break` leaves
  it; `for` runs its body once for each value of its source ([LOOP-3],
  [LOOP-4]), in order. The variable of a `for` is declared in the scope of
  the body, so it is not visible after the loop ([DECL-3]); it is immutable,
  and each iteration binds it to the next value. `for x: T in …` writes its
  type; `for _ in …` has no variable.
- **[LOOP-2]** `break` leaves the innermost loop whose body contains it, and
  `continue` ends the current iteration of that loop: `while` evaluates its
  condition again, `for` goes to its next value, `loop` runs its body again.
  A `break` or `continue` in a `match` arm leaves the loop around the
  `match`. The condition of `while` and the source of `for` (each bound of a
  range) are not in the loop's body: a `break` or `continue` there would act
  on the loop around it, and is an error in this version (accepting it later
  only accepts more programs). `break` and `continue` outside the body of
  every loop are errors. There are no labels and no `break` with a value in
  this version (OPEN #49).
- **[LOOP-3]** A range `a..b` is the integers from `a` up to `b`, `b`
  excluded, and is empty when `a ≥ b`; `a..=b` includes `b` and is empty when
  `a > b`. Both bounds have the same integer type, which each takes from the
  other ([TY-4]) or from the type of the variable (`for i: u8 in 0..n`);
  when neither gives one, the type is `usize`. `a` and then `b` are evaluated
  once, before the first iteration; assigning a variable that a bound read
  does not change the iterations. Iterating never overflows and never traps:
  after its last value the loop ends without computing another, so
  `for i: u8 in 250..=255` runs six times.
- **[LOOP-4]** `for x in s` iterates the elements of a slice `s` (a slice
  parameter or local, `&a`, `t.bytes`: any expression of type `&[T]`) or of
  an array local named `s`, which is the same as `&s` and requires the same
  ([REF-1]): an array that is not `mut`. The source is evaluated once; each
  iteration binds `x` to the next element, from the first to the last: a
  copy of it for a copy type, and *(decision 0020)* a borrow of it, for one
  iteration, for a type that moves ([OWN-5]). A slice views an immutable array, so nothing changes its elements or
  its length while the loop runs, and iterating needs no check at run time.
  A `mut` array is iterated by index (`for i in 0..a.len { … a[i] … }`).
  Text is not iterated ([TEXT-7]); its bytes are, `for b in t.bytes`. Other
  values (integers, `Option`, structs, an array that is not named) are not
  iterated.
- **[LOOP-5]** A statement **goes on** when execution can continue after it.
  `return`, `break` and `continue` never go on; an expression goes on when it
  produces a value; an `if` goes on when one of its branches does (a missing
  `else` does); a block goes on when all its statements do. A `while` whose
  condition is the literal `true`, and a `loop`, go on only when a `break`
  in their body leaves them; any other `while`, and every `for`, may go on.
  This rule decides [FN-2] (a function whose return type is not `()` must not
  go on at its end), unreachable statements (after a statement that does not
  go on) and [FN-3]: a block that does not go on has no value, and a `match`
  arm block that goes on has type `()`. So `while true { if c { break } }`
  goes on, and a function that returns a value needs a `return` after it.
- **[LOOP-6]** Loops are statements of `<- @eval` and `-> @effect`, at any
  depth ([CELL-2]); `break` and `continue` follow the region of their loop.
  They leave a loop, never a region or the cell.
- **[LOOP-7]** A loop has no effect of its own: the effects of its body are
  those of its statements ([EFF-3]), and running for ever is not an effect
  ([EFF-1]). A range or a slice is iterated without checks, so iterating has
  no trap and no site ([TRAP-3]); the operations of the body keep theirs.

## 6. Evaluation

### 6.1 Order

Operands and arguments are evaluated left to right, each completely, before the
operation or call. `&&` and `||` evaluate their right operand only when needed.
`match` evaluates its subject once and tries arms from top to bottom.

- **[ORD-1]** `a[i]` evaluates `i`, then checks bounds, then reads.
- **[ORD-2]** `a[i] = value` evaluates `i`, then checks bounds, then evaluates
  `value`, then stores. An out-of-bounds index stops the program before `value`
  is evaluated.
- **[ORD-3]** An array literal `[e1, e2, …]` evaluates every element, left to
  right, before the array is stored; an element may read the array being
  assigned.
- **[ORD-4]** `[value; N]` evaluates `value` exactly once, also when `N` is 0,
  and copies it into every element. *(decision 0020)* For a type that moves,
  this is a written form of copying: `value` moves into it, and the other
  elements are copies, which the tools report ([OWN-2]).
- **[ORD-5]** A struct literal evaluates its field values in the order they
  are **written**, not the order the fields are declared, each completely,
  before the struct value exists. A trap, or a `?` that returns, in one field
  value prevents the evaluation of the fields written after it.
- **[ORD-6]** `v.f = value` evaluates `value`, then stores it into the field;
  `value` sees the contents of `v` before the assignment. Likewise a literal
  assigned to `v` may read `v` (`p = Point { x: p.y, y: p.x }` swaps).

### 6.2 Integer arithmetic

**[NUM-1]** An **arithmetic** operation (`+ - * / %`, unary `-`) stops the
program (a *trap*) exactly when its mathematical result is not representable
in the operand type, or is undefined. Shifts and bitwise operations are
defined differently ([NUM-3], [NUM-4]): a shift traps only on an invalid
amount, and `<<` discards the bits shifted out by definition. The same rules
hold in every build mode; there is no unchecked release mode.

| Operation | Traps when | Trap kind |
|---|---|---|
| `a + b`, `a - b`, `a * b` | result outside the type's range | integer overflow |
| `-a` (signed) | `a` is the minimum value | integer overflow |
| `a / b` | `b == 0` | division by zero |
| `a / b` (signed) | `a` is the minimum value and `b == -1` | integer overflow |
| `a % b` | `b == 0` | division by zero |
| `a % b` (signed) | never for minimum / −1: the result is `0` | — |
| `a << n`, `a >> n` | `n` ≥ the bit width, or `n` negative | shift amount out of range |
| `a[i]` | `i ≥ len` | index out of bounds |

- **[NUM-2]** Division truncates toward zero; the remainder has the sign of the
  dividend (`-7 / 2 == -3`, `-7 % 2 == -1`, `7 % -2 == 1`).
- **[NUM-3]** `<<` discards the bits shifted out. `>>` is arithmetic for signed
  types and logical for unsigned types. Both operands have the same type.
- **[NUM-4]** `& | ^` operate on the two's-complement bits and never trap.
- **[NUM-5]** Comparisons compare mathematical values. `==` and `!=` also apply
  to `bool` and to enums whose variants hold no data; `Option`, `Result` and
  enums with data are inspected with `match` ([ENUM-4]); structs have no
  comparison operators ([STRUCT-7]). Floats compare as IEEE 754 says
  ([FLT-9]).

### 6.3 Traps

- **[TRAP-1]** A trap writes one report to standard error and ends the
  process with exit status **101**. The report names the kind of failure (the
  table of §6.2, a refused console write, [CON-4], or *(decision 0019)* a
  float-to-integer conversion out of range, [FLT-8]) and the position of the
  checked operation,
  `<file>:<line>:<column>`; by default it is one line of text,
  `yxy: trap[<code>]: <kind> at <file>:<line>:<column> (site <n>)` *(the code
  and site are experimental, [TRAP-3])*. It runs no cleanup and is not
  recoverable. Output written before the trap is kept (the test hooks write
  without buffering). *(experimental)* When several threads trap at the same
  time — threads of foreign code calling `export fn`, the only threads in
  this version — the traps of one linked image (an executable or a shared
  library, with every Yxy object linked into it) write one report: the first
  writes its report and ends the process, and the others write nothing. The
  scope is the linked image, not the process: Yxy code in two shared
  libraries of one process forms two images, and each may write one report
  before the process ends.
- **[TRAP-2]** Not a trap in this version: stack exhaustion from deep recursion
  or large arrays, and non-termination. The generated code requests stack
  probes, as clang does for C on this target, so that a large frame touches the
  guard page and the operating system ends the process instead of memory being
  overwritten. This is verified by a test that exhausts the stack in a child
  process and by inspecting the generated code.
- **[TRAP-3]** *(experimental)* Every failure this specification defines as a
  trap has defined behaviour up to the point where it is reported: the check
  comes before the operation it guards, and the compiler never turns the
  failing operation into undefined behaviour of its backend ahead of the check
  (for example by declaring that an addition cannot overflow or that an index
  is in bounds). Optimization may not remove a check whose failure is
  possible, move it after effects that follow it in evaluation order ([ORD-1]
  to [ORD-6]), or merge the reports of two checks. The report carries:
  - a **stable code** for the kind of failure, which is never reused for
    another kind;
  - the **site**: a number that identifies the check within the program, the
    same for a given source and compiler whatever the optimization level or
    the target;
  - the **source position** of the checked operation (file, line, column).

  The report has a human form and a structured (JSON) form carrying the same
  values, selected by the environment of the running program; their exact
  text, the codes and the numbering of sites are the compiler's contract,
  documented in the compiler repository (`docs/diagnostics.md`), like its
  diagnostic codes. Symbols that the hosted runtime uses to produce the report
  (`getenv`, besides `write` and `_exit`) are reserved as in [ABI-3].

### 6.4 Explicit arithmetic

| Operation | Result |
|---|---|
| `checked_add(a, b)`, `checked_sub`, `checked_mul` | `Some(result)`, or `None` when not representable |
| `checked_div(a, b)` | `None` when `b == 0` or signed minimum / −1 |
| `checked_rem(a, b)` | `None` when `b == 0`; `checked_rem(MIN, -1) == Some(0)` |
| `wrapping_add(a, b)`, `wrapping_sub`, `wrapping_mul` | result modulo 2^bits |

Both operands have the same integer type, taken from them or from context.

### 6.5 Conversions

- **[CONV-1]** `widen(x)` converts an integer to the integer type expected by
  context **only when every value fits on every supported target** (same
  signedness and not narrower, or unsigned to a strictly wider signed type,
  with `usize`/`isize` counted as both 32 and 64 bits). So `widen(u32 → usize)`
  and `widen(usize → u64)` are accepted, and `widen(u64 → usize)` is not.
  Anything else is an error at compile time.
- **[CONV-2]** `checked_convert(x)` converts to the `Option<T>` expected by
  context: `Some(v)` when the value fits in `T`, `None` otherwise. It is
  checked at run time.
- **[CONV-3]** There is no cast that drops the high bits of an integer (C's
  narrowing conversion). `as` is reserved. Floats convert with the
  operations of §6.6 ([FLT-7], [FLT-8]).

### 6.6 Floating point *(experimental, decision 0019)*

- **[FLT-1]** `f32` and `f64` are IEEE 754 binary32 and binary64, the same on
  every target. They are value types ([TY-1]): locals, parameters, results,
  struct fields, array elements, `Option` and `Result` payloads, the values
  of enum variants ([ENUM-1], decision 0017), and the C boundary ([ABI-2]). Their layout is their width, aligned as the target's
  data layout says (`f64` is 4-aligned on 32-bit x86 Linux, 8-aligned on the
  other targets; [STRUCT-8]).
- **[FLT-2]** A float literal ([LEX-19]) takes the float type its context
  expects ([TY-4]); with no context it is an error. Its value is the decimal
  value it writes, rounded to nearest, ties to even, in that type. A literal
  whose rounded value is infinite, or that is not zero but rounds to zero, is
  an error. A `-` directly applied to a float literal is part of it: `-0.0`
  is negative zero. There are no literals for infinities and NaN.
- **[FLT-3]** Floating point does not depend on the target: the same program
  gives the same bits on every target, but for the sign and payload of a NaN
  ([FLT-9]). Where the default of the target's C compiler would not
  guarantee it, the compiler fixes the CPU of the code it generates: on
  32-bit x86 Linux, a CPU with SSE2 (`pentium4`), so that `f32` and `f64`
  arithmetic is computed at its own precision and never in the x87 unit's
  extended precision; CPUs without SSE2 are not supported. Yxy code runs
  with round-to-nearest-even, no trapped floating-point exception and
  subnormal values kept, and on 32-bit x86 with the x87 precision control at
  its default (a 64-bit significand); the runtime never changes this, and
  foreign code must leave it so ([EFF-5]).
- **[FLT-4]** `a + b`, `a - b`, `a * b`, `a / b` apply to two operands of the
  same float type and give that type; `-a` negates. Each operation is the
  IEEE 754 operation, rounded once to nearest with ties to even. They
  **never trap**: an overflow gives an infinity, `x / 0.0` a signed infinity,
  `0.0 / 0.0` and `inf - inf` NaN, and no exception flag can be observed.
  Evaluation is **strict**: operands left to right, each operation rounded,
  grouped as written, so `a * b + c` rounds twice. The compiler does not fuse
  (contract), reassociate or otherwise change floating-point operations,
  applies no identity that changes a result (`x + 0.0` is not `x` for `-0.0`,
  `x * 0.0` is not `0.0` for an infinity), and assumes no value finite or
  not NaN — by default and under every option of this version; an optimizer
  may only make changes that keep every bit of every result, but the sign
  and payload of a NaN. Zeros have signs: every operation gives the sign
  IEEE 754 specifies (`x - x` is `+0.0`, `-0.0 + -0.0` is `-0.0`, `1.0 /
  -0.0` is negative infinity), and `-x` changes the sign of every value, zero
  and NaN included. A reduction whose order may change is not something the
  compiler does; it would come from an explicit interface (`decisions/OPEN.md`
  #8, after #43).
- **[FLT-5]** `%` is not defined for floats, nor are the bitwise operators,
  the shifts and `!`: each is an error.
- **[FLT-6]** `fma(a, b, c)` is `a * b + c` rounded **once** (IEEE 754
  fusedMultiplyAdd), for three operands of one float type, taken from them or
  from context as for §6.4. It is the one way to fuse a multiplication and
  an addition, and gives the same bits on every target.
- **[FLT-7]** `to_float(x)` converts an integer or a float to the float type
  expected by context, rounded to nearest, ties to even: exact from `f32` to
  `f64`; from `f64` to `f32` a value beyond the range of `f32` becomes an
  infinity of its sign, and NaN stays NaN. There is no implicit conversion
  between floats and integers or between `f32` and `f64`, and `widen` and
  `checked_convert` convert integers only ([CONV-1], [CONV-2]).
- **[FLT-8]** `truncate_to_int(x)` converts the float `x` to the integer type
  expected by context, rounding toward zero. When `x` is NaN or the rounded
  value does not fit the type (infinities included) the program **traps**
  with the kind *float-to-integer conversion out of range* ([TRAP-1],
  [TRAP-3]). `checked_truncate_to_int(x)` gives `Some(v)` instead, or `None`
  where the other traps; it never traps.
- **[FLT-9]** `==`, `!=`, `<`, `<=`, `>`, `>=` on floats compare as IEEE 754
  does: `-0.0 == 0.0` holds, and with a NaN operand every comparison is false
  but `!=`, which is true (`x != x` holds exactly when `x` is NaN). Equality
  is not a comparison of bits. An operation whose operands are not NaN never
  produces a signaling NaN; the NaN it produces is quiet. A signaling NaN can
  only come from foreign code; an operation with a signaling operand gives a
  NaN, quiet or not (not specified: a value computed when compiling and one
  computed when running may differ), `-x` keeps the quietness of `x`, and
  passing a NaN through the C boundary may make it quiet. The **sign and
  payload of a NaN are not specified** either (they differ between targets
  and between a value computed when compiling and one computed when
  running), and no operation of this version observes them.
- **[FLT-10]** A float literal is never a pattern: `match` arms of floats are
  `_` or a binding, and a `match` on a float needs one of them to cover every
  value ([MATCH-1]).

## 7. Effects

- **[EFF-1]** `effects { … }` lists the effects a function may perform.
  `effects {}` means **none of the effects tracked by the language**. It does
  not mean that the function cannot trap, always terminates, reads no
  arguments, or costs nothing.
- **[EFF-2]** Tracked effects in this version: `ffi` — calling code outside
  Yxy — and *(experimental, decision 0015)* `console` — writing through a
  `Console` capability ([CON-3]). Other names are errors. The set will grow as
  the standard library appears. Which other concerns become effects is not
  settled: files, network, clock and randomness are candidates, and allocation has facets of
  an effect, a capability and a type, to be decided with ownership
  (`decisions/OPEN.md` #6). *(experimental)* The criterion of
  `decisions/OPEN.md` #9 (type, contract, effect or capability) is applied to
  each candidate.
- **[EFF-3]** At every call, the callee's declared effects must be a subset of
  the caller's declared effects. Because the rule uses declared effects, it
  holds through recursion. Where the call appears (which region) does not
  matter: `@effect` grants nothing, and `@eval` may call declared effects.
  Across packages, an imported function's effects are those of its public
  signature (`modules.md` [VIS-5]).
- **[EFF-4]** An `extern fn` must declare `ffi`. The effects of foreign code are
  a **trusted declaration**, not verified; foreign calls are a trust boundary
  and tools report them as such.
- **[EFF-5]** `ffi` is also the **unsafe boundary** of this version: foreign
  code can do anything with the integers it receives, including treating them
  as addresses. The memory-safety guarantees of this specification hold for
  code whose effects exclude `ffi`, and for the Yxy side of every call. There is
  no `unsafe` construct yet. *(experimental)* These guarantees assume that the
  foreign side keeps the contract of the boundary: foreign callers of an
  `export fn` follow the target's C ABI ([ABI-2]); foreign code does not
  unwind across Yxy frames ([ABI-3] (c)); *(decision 0019)* foreign code
  leaves the floating-point environment as Yxy code runs it —
  round-to-nearest-even, no trapped exception, subnormals kept, and on
  32-bit x86 the x87 precision control at its default, a 64-bit significand
  ([FLT-3]) — whenever control returns to Yxy code or enters it; and the objects and C
  files linked into the program (`--link`) do not define the C symbols that
  the generated code calls: the hosted runtime's `write`, `_exit` and
  `getenv`, helpers of the C implementation whose names start with `__`
  ([ABI-3] (a): the stack probe, the arithmetic helpers of 32-bit targets),
  and *(decision 0019)* the C library's `fma` and `fmaf`, which carry out
  [FLT-6] on a CPU without a fused multiply-add instruction. The code generated to copy
  a struct or to fill an array with `[value; N]` calls no function that the
  program or a linked object can define, so a linked object that defines
  `memcpy` or `memset` does not reach it, and it has no effect of its own,
  also in a function declared `effects {}`. The compiler rejects an
  `export fn` with a reserved name, but it does not check what the linked
  objects define.

| Phenomenon | Classification |
|---|---|
| call to a Yxy function | the callee's declared effects |
| call to an `extern fn` | `ffi` plus its declared effects (trusted) |
| operation of a `Console` (`console.print(…)`) | `console` ([CON-3]) |
| operators, indexing, `.len`, `.bytes`, text equality, §6.4, §6.5, §6.6 | no tracked effect; may trap (§6.2, [FLT-8]; float arithmetic never traps, [FLT-4]) |
| loops (`while`, `loop`, `for`) | no tracked effect; iterating never traps ([LOOP-7]) |
| trap | not an effect; it writes its message to standard error and ends the process |
| typed failure (`Result`, `require`, `?`) | not an effect; it is in the return type |
| mutation of a local `mut` variable | not an effect |
| allocation | does not exist in this version |

Static effect checking is not an operating-system sandbox.

### 7.1 The console capability *(experimental, decision 0015)*

A capability is a value that carries authority over a resource; an effect
describes what may happen. The console has both: the capability says which
stream, the effect that a write may happen.

- **[CON-1]** `Console` is a capability: the authority to write to the
  process's standard output. No expression creates one: the runtime of a
  hosted program gives it to `main` ([MAIN-1]), and a function that writes to
  it receives it as a parameter. There is no global console: there are no
  global variables ([INIT-1]). A `Console` may be a parameter or a local
  variable; passing or copying it copies the authority, not the stream. It
  cannot be returned or stored in a struct, an array, `Option` or `Result`,
  and does not cross the C boundary, so that the parameters of a function
  show every console it can reach.
- **[CON-2]** Operations, called with the capability before `.` ([GR-1]):
  `console.print(text: &str)` writes the bytes of the text and nothing else
  (no line break); `console.print_u64(value: u64)` and
  `console.print_i64(value: i64)` write the value in decimal, `-` before a
  negative one, with no padding. Each returns `()`. `Console` has no other
  operation.
- **[CON-3]** Each operation performs the effect `console`: the function that
  calls one must declare it, and [EFF-3] carries it to every caller.
  Holding a `Console` without declaring `console` gives no way to write, and
  declaring `console` without receiving a `Console` gives none either.
- **[CON-4]** An operation writes all its bytes before it returns, in order
  and without buffering, so that output written before a trap is kept
  ([TRAP-1]); empty text writes nothing. When the stream refuses a write
  (it is closed for writing, a device fails, the disk is full, or a signal
  interrupts the write) the program traps with the kind *console write
  failed* at the operation ([TRAP-1], [TRAP-3]). The operations are not
  recoverable and are documented as trapping. On a pipe whose reader has
  gone, the operating system's default for SIGPIPE applies (it ends the
  process); a program does not change signal dispositions.

## 8. The C boundary

- **[ABI-1]** `extern fn name(…) -> T effects { ffi }` declares a C function
  defined outside Yxy. `export fn` defines a Yxy function callable from C under
  its own name. All other functions are internal to the program. `export`
  names are unique in the whole program, and `extern` declarations of one
  symbol in different packages must agree (`modules.md` [VIS-8]).
- **[ABI-2]** Only integers, `bool` and *(decision 0019)* the floats cross
  the boundary (and `()` as a return type); `f32` is C's `float` and `f64`
  C's `double`, on every target. Slices, enums (with or without data),
  structs, `Option` and `Result` do not, because their layout is not a
  stable ABI ([STRUCT-8], [ENUM-5]). Integers and `bool` narrower than
  32 bits follow the C ABI of each target, which differ: some make the caller
  extend them to 32 bits (Apple's ARM64 ABI), some leave the bits beyond the
  value's width unspecified (the generic AAPCS64). When Yxy passes such a
  value to C — an argument of an `extern fn`, the result of an `export fn` —
  it extends it as the target's ABI requires. *(experimental)* When Yxy
  receives one from C — a parameter of an `export fn`, the result of an
  `extern fn` — it does not rely on the bits beyond the value's width: the Yxy
  side normalizes the value at the boundary, so that Yxy code always sees a
  value of the declared type (an integer in its type's range, a `bool` that is
  `false` or `true`), on every target, whatever those bits hold.
- **[ABI-3]** Reserved symbols, the same on every target ([TGT-2]), including
  a target that never uses a given name:
  - (a) the symbols of the hosted runtime — `main`, `write`, `_exit`,
    `getenv` —, names starting with `yxy_rt_`, and names starting with `__`
    (reserved for the C implementation, such as the stack probe
    `__chkstk_darwin` and the arithmetic helpers of 32-bit targets) cannot be
    `extern` or `export` symbols;
  - (b) *(experimental)* the C library functions that the generated code may
    call without a call in the source, to copy, fill or compare memory —
    `memcpy`, `memmove`, `memset`, `memcmp`, `bcmp`, `bzero` and
    `memset_pattern16` — and *(decision 0019)* to carry out `fma` of `f32`,
    `fmaf` (`fma`, of `f64`, is a prelude name, [PRG-2]), and names starting with `_` (C11 7.1.3 reserves
    them for the C implementation as identifiers with file scope, which an
    `export` symbol is: `_start`, the entry point of ELF programs,
    `_mh_execute_header` of Mach-O executables, `_GLOBAL_OFFSET_TABLE_` of
    32-bit x86 objects), cannot be `export` symbols: an `export fn` with one
    of these names would replace it for the whole program, or fail at link
    time. An
    `extern fn` may declare them; it only names the C library's function, and
    calling it is an ordinary `ffi` call. The code generated for copies and
    fills does not call these functions ([EFF-5]); the reservation is a second
    defence. Whether more names of the C library should be reserved is open
    (`decisions/OPEN.md` #44);
  - (c) *(experimental)* unwinding: foreign code must not unwind into or
    across Yxy frames — a C++ exception, a forced unwind (the end or the
    cancellation of a thread where the C library implements it by
    unwinding), a `longjmp` or any other non-local exit that passes over a
    Yxy function. This is a precondition of the `ffi` trust boundary
    ([EFF-4], [EFF-5]), which the compiler relies on: Yxy functions and
    `extern` declarations are compiled as never unwinding. C++ code called
    from Yxy catches its exceptions before it returns (for example in a
    `noexcept` wrapper with `catch (...)`). When the precondition is broken
    through the platform's unwinder (Itanium C++ ABI), the frame of the Yxy
    function that called the foreign code refuses the unwind when the
    unwinder reaches it, and the process ends (for a C++ exception, the C++
    runtime calls `std::terminate`) instead of skipping Yxy frames; this
    holds where that frame is on the stack while the foreign code runs. An
    exit that does not go through the unwinder (a `longjmp`, the end of a
    thread that does not unwind) is not detected; what it does, like an
    unwind that no Yxy frame refuses, is outside the guarantees of this
    specification. Whether the end of the process should become a trap
    report ([TRAP-1]), or unwinding a declared contract, is decided together
    with destructors (`decisions/OPEN.md` #45).

## 9. Targets

- **[TGT-1]** A program is compiled for one target, chosen explicitly
  (default: the host). The target fixes the data model ([TY-2]), the internal
  layout of structs ([STRUCT-8]), the generated code and the runtime; it does
  not change syntax, names, other types, effects or cells.
- **[TGT-2]** A program's validity depends on the target only through the
  range of `usize`/`isize` values (for example, the literal `4294967296` does
  not fit `usize` on a 32-bit target) — never through conversion rules
  ([CONV-1]). *(decision 0018)* The same holds for the values of constants
  and the lengths of arrays computed at compile time ([CONST-6]): one that
  overflows `usize` on a 32-bit target only is an error there only. A
  warning ([CONST-7]) may depend on the target and never decides validity.
- **[TGT-3]** `usize` crosses the C boundary as `size_t` of the target.

## 10. Entry point

- **[MAIN-1]** `fn main()` takes no parameters, or *(experimental, decision
  0015)* one parameter of type `Console`, `fn main(console: Console)` (any
  name), which the runtime of a hosted program gives: the console of
  standard output ([CON-1]). It returns `()` (exit status 0) or `u8` (the
  exit status), and declares its effects like any function. How the other
  capabilities (files, network, clock, randomness) reach `main` is future
  work (`decisions/OPEN.md` #9).

## 11. Incomplete programs

- **[HOLE-1]** A typed hole `$` or `$name` stands for a missing expression. The
  compiler reports each hole with the type expected at that position when the
  context gives one (and says when it does not), in its human and structured
  diagnostics; `check` and `build` reject any program that still contains a
  hole. *(Scope in this version: the expected type is part of the
  diagnostic's text, not a field of its own, and a program that still contains
  a hole has no facts (`inspect`) for its other functions either. A
  structured description of holes for tools and agents — the expected type as
  data, the names in scope, facts of the rest of the program — is future
  work.)*

## 12. Outside this version

Rejected with a diagnostic, never ignored: `when`, named fields in enum
variants (`V { a: T }`), recursive enums, `==` on enums with data, rest
patterns (`..`) and field shorthand in patterns, destructuring declarations
(`P { x: a } := p`), user-defined generics, traits, closures, function
values, method calls (other than the operations of a `Console`, [CON-2],
and `.copy()`, [OWN-6]), `inout` parameters, destructors and owned
resources (decision 0020, TASK-20260926-042), an implicit copy of a struct
or of an enum with data (decision 0020, [OWN-1]),
loop labels, `break` with a value,
ranges as values, ranges without a bound, iterating a `mut` array or text by
element (OPEN #49), compound assignment (`+=` and the other `op=` forms),
dependencies on other modules, manifest requirements,
the lock and fetching (specified in `modules.md`, not implemented), `unsafe`, `&mut`, references other than slices,
arrays as parameters or return values, nested cells, `if` as an expression
([GR-3]), or-patterns, match guards and range patterns (OPEN #46), sub-slices
(OPEN #47), indexing or `.len` of an array or slice that is not named
([REF-3], [REF-4]),
owned strings, characters, the text operations of [TEXT-7], the float
operations of [FLT-5], float literals as patterns ([FLT-10]), floating-point
types other than `f32` and `f64`, printing floats (`decisions/OPEN.md` #8),
128-bit integers, concurrency (`par`,
`async`), casts (`as`), block comments, *(decision 0018)* constants inside
a function, constants of types other than the integers and `bool`, calls in
constant expressions other than `widen` (`const fn`), constants as patterns, generic enums, mutable slices, enums
without variants and enums with more than 256 variants; for structs: generic
structs, structs without fields, arrays and slices as fields, recursive
structs, structs beyond [STRUCT-9], equality, methods, field
shorthand, update syntax, fields of array elements or temporaries as assignment
targets, and structs at the C boundary.

## Glossary

Terms whose meaning this specification fixes where ordinary usage varies.

- **truncate** *(decision 0019)*: to round toward zero, dropping the
  fraction of a value: integer division truncates ([NUM-2]), and
  `truncate_to_int` truncates a float to an integer ([FLT-8]). It never
  means dropping the high bits of an integer to fit a narrower type (the
  narrowing conversion of C, which some languages call truncation): no
  operation of the language does that ([CONV-3]).
