# Yxy semantics — experimental, slice 1

Status: **experimental**. Normative for the first implemented subset. Syntax is
in `syntax.md`; decisions with alternatives and risks are in `decisions/`.
"Error" below means the compiler rejects the program with a diagnostic. The
compiler's diagnostic codes are listed in the compiler repository
(`docs/diagnostics.md`); they are tooling, not part of the language.

## 1. Programs

- **[PRG-1]** A program is one source file, which is one module. It starts with
  `module <name>`. Imports are not supported yet.
- **[PRG-2]** Items are enums and functions. Item names are unique in the
  module. The prelude names — `bool`, the integer type names, `Option`,
  `Result`, `Some`, `None`, `Ok`, `Err`, the operations in §6.4 and §6.5 — can
  not be redefined, neither as items nor as parameters or local variables.
- **[PRG-3]** An executable program has `fn main()` (§9). A file without
  `main` can be compiled to an object and linked with other code.

## 2. Types

| Type | Values | Where it may appear |
|---|---|---|
| `bool` | `true`, `false` | anywhere |
| `i8 i16 i32 i64 isize` | two's-complement integers of that width | anywhere |
| `u8 u16 u32 u64 usize` | unsigned integers of that width | anywhere |
| `()` | the single unit value `()` | anywhere; a function without `-> T` returns `()` |
| `E` (an `enum`) | one of its variants, written `E.Variant` | anywhere |
| `Option<T>` | `None` or `Some(v)` | anywhere; `T` is a value type |
| `Result<T, E>` | `Ok(v)` or `Err(e)` | anywhere; `T`, `E` are value types |
| `[T; N]` | exactly `N` elements | local variables only, created from a literal (`[a, b]` or `[v; N]`) |
| `&[T]` | a read-only view of an array's elements | parameters and local variables only |

- **[TY-1]** *Value types* are `bool`, the integers, `()`, enums, and `Option`
  and `Result` of value types. They are copied on assignment and when passed.
- **[TY-2]** `usize` and `isize` have the pointer width of the target's data
  model: 32 or 64 bits in this version. They are distinct types from `u32`,
  `u64`, `i32` and `i64` on every target. All other integer types have the
  same width on every target.
- **[TY-3]** Enum variants carry no data in this version, and an enum has
  between 1 and 256 variants. `Option` and `Result` are known to the compiler;
  user-defined generics do not exist yet.
- **[TY-4]** There are **no implicit conversions**, and **no default integer
  type**. An integer literal takes the integer type expected by its context
  (annotation, parameter, the other operand, the return type…). With no
  context, it is an error. A literal that does not fit its type is an error;
  a `-` directly applied to a literal is part of the literal, so
  `x: i8 := -128` is valid.
- **[TY-5]** `None`, `Ok(…)` and `Err(…)` need an expected type. `Some(v)` can
  take its type from `v`.
- **[TY-6]** Arrays are created only from literals. Copying a whole array from
  another array (`b := a`) is not supported yet, so no buffer is ever copied
  silently; an array is read elsewhere through a borrow (`&a`). A `match` does
  not produce arrays or slices.

## 3. Names, declarations and mutability

- **[DECL-1]** `name := value` declares an **immutable** local variable.
  `mut name := value` declares a mutable one. `name: T := value` and
  `mut name: T := value` add a type annotation. `_ := value` evaluates and
  discards a value. `let` and `var` do not exist.
- **[DECL-2]** `place = value` assigns. The place is a `mut` local variable or
  an element `a[i]` of a `mut` array. Slices are read-only and cannot be
  reassigned.
- **[DECL-3]** Blocks (`{ … }` of `if`, `while` and match arms) open scopes.
  **Shadowing is not allowed**: a name cannot be declared while another
  declaration with the same name is visible in the function, and a local or a
  parameter cannot take the name of an item.
- **[DECL-4]** The regions of a cell form one sequential scope: a name declared
  in a region is visible in the following regions, from its declaration on.
- **[DECL-5]** Every value is used explicitly: an expression statement whose
  value is not `()` is an error; write `_ := expr` to discard.
- **[DECL-6]** In a pattern, a bare name binds the value. When the matched type
  is an enum that has a variant with that name, the pattern is an error (it
  would silently match everything); the variant is written `Enum.Variant`.

## 4. Borrows

- **[REF-1]** `&a`, where `a` is an **immutable** array local, produces a slice
  `&[T]` over its elements. Borrowing a `mut` array, a scalar or a slice is an
  error. Because the borrowed array is immutable, a slice never observes a
  mutation.
- **[REF-2]** References cannot escape: no function returns a slice, and slices
  cannot be stored in arrays, `Option` or `Result`. A slice parameter or local
  therefore never outlives the array it views. This is a restriction of the
  subset, not a borrow checker; wider borrowing rules are future work.
- **[REF-3]** `s.len` is the number of **elements** of an array or slice, of
  type `usize`.
- **[REF-4]** `a[i]` requires `i: usize` and checks `i < len` at run time (§6.1).
- **[REF-5]** All types of this version are copied; there are no move-only
  values yet, so use-after-move cannot occur.

## 5. Functions and cells

- **[FN-1]** A function declares parameter types, its return type (or `()` when
  omitted) and its effects: `effects { … }` is mandatory (§7).
- **[FN-2]** A function whose return type is not `()` must return on every path.
  Statements that can never run (after a `return`, or regions after one that
  always returns) are errors.
- **[FN-3]** An expression that never produces a value (a `match` whose arms
  all return) can only be used as a statement; binding it, passing it or
  returning it is an error.
- **[MATCH-1]** A `match` covers every value of the matched type; otherwise it
  is an error that names one value not covered. Integer matches need a final
  `_` arm.
- **[MATCH-2]** An arm that can never match, because earlier arms cover every
  value it matches, is an error.
- **[MATCH-3]** Without an expected type, the type of a `match` comes from its
  arms: an arm whose value needs a type from context (an integer literal,
  `None`) takes it from the other arms, whatever their order.
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
- **[CELL-2]** *(experimental)* Statements allowed in each region:
  `@ctrl` — declarations and `require`; `@eval` and `@effect` — declarations,
  assignments, `if`, `while` and calls; `@out` — declarations, `if`, calls and
  `return`. Consequently validation (`require`) happens only in `@ctrl`, and an
  early exit before `@out` happens only through `?`. See
  `decisions/0004-cell-regions.md` for the alternative (no restriction) and
  what would change it.
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
- **[CELL-6]** Failures, by region: a false guard in `@ctrl` prevents the
  evaluation and every effect; a failure in `@eval` prevents the effect steps;
  a failure in an effect step does **not** undo the actions already performed
  and prevents the later steps. There is no implicit rollback, retry or
  exactly-once delivery; transactions and idempotence belong to explicit
  library contracts.
- **[CELL-7]** `return` leaves the function. `when` (a cell that is skipped when
  its condition is false) is reserved and not supported yet.
- **[CELL-8]** No value in this version owns a resource, so leaving a function
  runs no cleanup code. Deterministic cleanup order is future work.

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
  and copies it into every element.

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
  to `bool` and enums; `Option` and `Result` are inspected with `match`.

### 6.3 Traps

- **[TRAP-1]** A trap writes one line to standard error,
  `yxy: trap: <kind> at <file>:<line>:<column>`, and ends the process with exit
  status **101**. It runs no cleanup and is not recoverable. Output written
  before the trap is kept (the test hooks write without buffering).
- **[TRAP-2]** Not a trap in this version: stack exhaustion from deep recursion
  or large arrays, and non-termination. The generated code requests stack
  probes, as clang does for C on this target, so that a large frame touches the
  guard page and the operating system ends the process instead of memory being
  overwritten. This is verified by inspecting the generated code; no test
  exhausts the stack yet.

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
- **[CONV-3]** There is no truncating cast. `as` is reserved.

## 7. Effects

- **[EFF-1]** `effects { … }` lists the effects a function may perform.
  `effects {}` means **none of the effects tracked by the language**. It does
  not mean that the function cannot trap, always terminates, reads no
  arguments, or costs nothing.
- **[EFF-2]** Tracked effects in this version: `ffi` — calling code outside
  Yxy. Other names are errors. The set will grow (for example console, files,
  network, clock, randomness, allocation) as the standard library appears.
- **[EFF-3]** At every call, the callee's declared effects must be a subset of
  the caller's declared effects. Because the rule uses declared effects, it
  holds through recursion. Where the call appears (which region) does not
  matter: `@effect` grants nothing, and `@eval` may call declared effects.
- **[EFF-4]** An `extern fn` must declare `ffi`. The effects of foreign code are
  a **trusted declaration**, not verified; foreign calls are a trust boundary
  and tools report them as such.
- **[EFF-5]** `ffi` is also the **unsafe boundary** of this version: foreign
  code can do anything with the integers it receives, including treating them
  as addresses. The memory-safety guarantees of this specification hold for
  code whose effects exclude `ffi`, and for the Yxy side of every call. There is
  no `unsafe` construct yet.

| Phenomenon | Classification |
|---|---|
| call to a Yxy function | the callee's declared effects |
| call to an `extern fn` | `ffi` plus its declared effects (trusted) |
| operators, indexing, `.len`, §6.4, §6.5 | no tracked effect; may trap (§6.2) |
| trap | not an effect; it writes its message to standard error and ends the process |
| typed failure (`Result`, `require`, `?`) | not an effect; it is in the return type |
| mutation of a local `mut` variable | not an effect |
| allocation | does not exist in this version |

Static effect checking is not an operating-system sandbox.

## 8. The C boundary

- **[ABI-1]** `extern fn name(…) -> T effects { ffi }` declares a C function
  defined outside Yxy. `export fn` defines a Yxy function callable from C under
  its own name. All other functions are internal to the program.
- **[ABI-2]** Only integers and `bool` cross the boundary (and `()` as a return
  type). Slices, enums, `Option` and `Result` do not, because their layout is
  not a stable ABI. Integers and `bool` narrower than 32 bits are extended by
  the caller, as the target ABI requires.
- **[ABI-3]** Symbols the generated code refers to — `main`, `write`, `_exit` —,
  names starting with `yxy_rt_` and names starting with `__` (reserved for the C
  implementation, such as the stack probe `__chkstk_darwin`) cannot be `extern`
  or `export` symbols. Unwinding across the boundary does not exist.

## 9. Targets

- **[TGT-1]** A program is compiled for one target, chosen explicitly
  (default: the host). The target fixes the data model ([TY-2]), the generated
  code and the runtime; it does not change syntax, names, other types, effects
  or cells.
- **[TGT-2]** A program's validity depends on the target only through the
  range of `usize`/`isize` values (for example, the literal `4294967296` does
  not fit `usize` on a 32-bit target) — never through conversion rules
  ([CONV-1]).
- **[TGT-3]** `usize` crosses the C boundary as `size_t` of the target.

## 10. Entry point

- **[MAIN-1]** `fn main()` takes no parameters and returns `()` (exit status 0)
  or `u8` (the exit status). It declares its effects like any function. How
  capabilities reach `main` in a hosted environment is future work.

## 11. Incomplete programs

- **[HOLE-1]** A typed hole `$` or `$name` stands for a missing expression. The
  compiler reports it with the type expected at that position, so tools and
  agents can inspect the gap; `check` and `build` reject any program that still
  contains a hole.

## 12. Outside this version

Rejected with a diagnostic, never ignored: `when`, structs, enum payloads,
user-defined generics, traits, closures, function values, method calls, `for`,
`loop`, `break`, `continue`, imports and modules, `pub`, `unsafe`, `&mut`,
references other than slices, arrays as parameters or return values, nested
cells, `if` as an expression, strings, characters, floating point, 128-bit
integers, concurrency (`par`, `async`), casts (`as`), block comments, generic
enums, mutable slices, enums without variants and enums with more than 256
variants.
