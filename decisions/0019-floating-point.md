# Decision 0019: Floating point

- Status: Accepted — **experimental** (proposed with TASK-20260926-038,
  which follows the research recommendation of the architecture audit of
  2026-09-26, tension T16, option (2); open to the author's review)
- Date: 2026-09-28
- Spec: `syntax.md` [LEX-11], [LEX-19], §3 (`primary`, patterns);
  `semantics.md` §2 (table), [PRG-2], [TY-1], [TY-4], [NUM-5], §6.6
  ([FLT-1]–[FLT-10]), [CONV-1], [EFF-5], the table of §7, [ABI-2], [ABI-3]
  (b), §12; `OPEN.md` #8 (rewritten) and #48 (names now reserved)
- Author requirements and sources it follows (register of the control
  repository and the audit): M-2 (nothing silent), M-4 (no abort hidden in an
  API presented as recoverable), the master prompt's rule that release
  builds do not change a program's meaning (l.210) and its deterministic
  results across targets (l.244), the constitution's list of what must be
  documented before 1.0 (NaN, infinity, signed zero, rounding, float to
  integer; l.475–488), and the audit's T16 evidence

## Context

Before this decision the language had no floating point: `f32` and `f64`
were refused as types and `1.5` as a literal (the compiler's E0900), and
`OPEN.md` #8 waited for "a policy for NaN, signed zero, rounding and no
fast-math by default".

Floating point is where "the same program gives the same result" breaks most
easily. The audit (T16) measured the ways:

- **Contraction.** The same C, `a * b + a`, compiles to `fmadd` on AArch64
  (one rounding), to `mulsd` and `addsd` on baseline x86-64 (two roundings)
  and to `vfmadd213sd` on x86-64-v3: C's default `-ffp-contract=on` lets the
  compiler fuse, and whether it does depends on the CPU. LLVM IR without
  the `contract` flag is never fused, whatever the driver's options; with
  the flag it is, even under `-ffp-contract=off`. The front end alone
  decides.
- **NaN bits.** `0.0 / 0.0` is `7ff8000000000000` on AArch64 and
  `fff8000000000000` on x86-64; LLVM's constant folder gives the positive
  one, so on x86 the same expression has different bits at `-O0` and at
  `-O2`.
- **i686.** Without SSE2, x87 computes in extended precision and rounds when
  it stores, so results differ from binary64 (double rounding, a wider
  exponent range); the i686 C ABI returns `float` and `double` in an x87
  register, and loading a signaling NaN there makes it quiet.
- **The reference evaluator** computes with IEEE operations; any freedom the
  generated code takes (fusion, reassociation, excess precision) makes the
  three engines disagree, and the differential testing of the compiler stops
  working for floats.

The audit recommended option (2): strict IEEE by default, an explicit fused
multiply-add, reorderable reductions only through an explicit interface
(after the *as-if* reading of `OPEN.md` #43), a minimum i686 CPU with SSE2
fixed by the compiler rather than inherited from clang's default, the
quietness of NaN specified, and fast-math never by default.

## Decision

1. **Types** ([FLT-1]). `f32` and `f64` are IEEE 754 binary32 and binary64,
   the same on every target. They are value types, like the integers: they
   can be locals, parameters, results, struct fields, array elements and
   `Option`/`Result` payloads, and they cross the C boundary ([ABI-2]). Their
   names join the prelude ([PRG-2]).
2. **Literals** ([LEX-19], [FLT-2], [TY-4]). A float literal is written
   `digits.digits`, with an optional exponent `e<digits>` or `e-<digits>`
   (`1.5`, `0.25`, `6.02e23`, `1.0e-9`); `_` separates digits as in integers.
   It has one spelling per value and digit grouping: `1.`, `.5`, `1e5`,
   `1.5E3`, `1.5e+3` and exponents with leading zeros are errors, each with a
   mechanical fix to the one spelling; there are no type suffixes (`1.5f32`)
   and no hexadecimal floats. A float literal has no type of its own: it
   takes the float type its context expects, like an integer literal, and
   with no context it is an error — there is no default float type. It is
   rounded to nearest, ties to even, in that type; a literal whose rounded
   value is infinite, or that is not zero but rounds to zero, is an error. A
   `-` applied to a literal is part of it, so `-0.0` is negative zero. An
   integer literal is never a float (`x: f64 := 1` is an error, fixed to
   `1.0`), and a float literal never an integer. There are no literals for
   infinities and NaN; operations produce them.
3. **Arithmetic** ([FLT-4]). `+`, `-`, `*`, `/` and unary `-` apply to two
   operands of the same float type and give that type. Each is the IEEE 754
   operation, rounded once to nearest with ties to even — the only rounding
   mode. They never trap: IEEE's default results stand (an overflow is an
   infinity, `x / 0.0` a signed infinity, `0.0 / 0.0` and `inf - inf` NaN),
   and the exception flags are not observable. Subnormal values are kept (no
   flush to zero). `-x` changes the sign of every value, zeros and NaN
   included, and is never computed as `0.0 - x` (which gives `+0.0` for
   `+0.0`).
4. **Strict evaluation** ([FLT-4]). An expression is evaluated as written:
   operands left to right, each operation rounded, grouping as the
   parentheses and precedence say. `a * b + c` is two roundings. No
   contraction, no reassociation, no algebraic identity that changes a
   result (`x + 0.0` is not `x` for `-0.0`, `x - x` is not `0.0` for an
   infinity or NaN, `x * 0.0` is not `0.0`), no reciprocal instead of a
   division, no assumption that values are finite or not NaN: none of the
   fast-math relaxations, by default or by any option of this version. An
   optimizer may only make changes that give the same bits, but for the sign
   and payload of a NaN ([FLT-9]). The generated code carries no fast-math
   flag (`contract`, `reassoc`, `nnan`, `ninf`, `nsz`, `arcp`, `afn`,
   `fast`). A reduction whose order may change (a vectorized sum) is not an
   optimization the compiler makes: it will only come from an explicit
   interface, after the *as-if* reading of [TRAP-3] is settled (`OPEN.md`
   #43, #8).
5. **`fma`** ([FLT-6]). `fma(a, b, c)` is `a * b + c` rounded once (IEEE 754
   fusedMultiplyAdd), for three operands of one float type, taken from the
   operands or from context like the operations of §6.4. It is the one way
   to ask for fusion, and it gives the same bits on every target: the
   generated code uses the CPU's instruction where the target's minimum CPU
   has one (AArch64), and otherwise the C library's correctly rounded `fma`
   or `fmaf` (C11 7.12.13.1) — the x86 targets, whose minimum CPUs have no
   fused multiply-add.
6. **Comparisons and equality** ([FLT-9]). `==`, `!=`, `<`, `<=`, `>`, `>=`
   compare values as IEEE 754 does: `-0.0 == 0.0` holds; with a NaN operand
   every comparison is false except `!=`, which is true (`x != x` holds
   exactly when `x` is NaN). Equality is IEEE's, not a comparison of bits.
   There is no total order in this version.
7. **NaN** ([FLT-9]). A NaN that an operation produces is quiet; no
   operation of the language produces a signaling NaN. The sign and the
   payload of a NaN result are **not specified**: they differ between
   targets and between the constant folder and the hardware (Context), and
   no operation of this version can observe them (there is no access to the
   bits and no printing of floats). A signaling NaN can only come from
   foreign code; operations make it quiet, and on i686 the C ABI's x87
   return may make it quiet too. A NaN converts to no integer ([FLT-8]).
8. **Signed zero** ([FLT-4], [FLT-9]). `+0.0` and `-0.0` are distinct values
   that compare equal; every operation gives the sign IEEE 754 specifies
   (`x - x` is `+0.0`, `-0.0 + -0.0` is `-0.0`, `1.0 / -0.0` is `-inf`).
9. **Conversions** ([FLT-7], [FLT-8]), explicit, by prelude functions that
   take the target type from context (as `widen` and `checked_convert` do):
   - `to_float(x)`: an integer or a float to the float type the context
     expects, rounded to nearest, ties to even: exact from `f32` to `f64`;
     from `f64` to `f32` a value beyond the range of `f32` becomes an
     infinity and NaN stays NaN; every integer of this version is within the
     range of `f32`.
   - `truncate_to_int(x)`: a float to the integer type the context expects,
     rounded toward zero; when `x` is NaN or the rounded value does not fit
     the type (infinities included) the program **traps** with the new kind
     "float-to-integer conversion out of range" (the compiler's `T0006`).
   - `checked_truncate_to_int(x)`: the same, returning `Option`: `None`
     instead of the trap. It checks nothing at run time.
   There is no implicit conversion between floats and integers or between
   `f32` and `f64`; `widen` and `checked_convert` stay integer conversions;
   `as` stays reserved ([CONV-3]).
10. **`%` is refused** ([FLT-5]), as are the bitwise operators, the shifts and
    `!` on floats.
11. **Patterns** ([FLT-10]). A float literal is never a pattern. A `match` on
    a float value is exhaustive only with `_` or a binding.
12. **Targets** ([FLT-3]). The same program gives the same bits on every
    target (NaN's sign and payload aside). The compiler fixes the CPU of the
    code it generates where the triple's default would not guarantee it: on
    `i686-unknown-linux-gnu` every function is compiled for `pentium4`, the
    first x86 CPU with SSE2, so `f32` and `f64` arithmetic runs in SSE2
    registers at its own precision, as on the 64-bit targets, never in x87
    registers with extended precision; x86 CPUs without SSE2 are not
    supported. The x87 unit remains only where SSE2 has no instruction or
    the C ABI requires it: a `float`/`double` result returned by a function
    with the C calling convention (a load and a store that keep every value,
    a signaling NaN aside), and conversions between 64-bit integers and
    floats, which it computes exactly (64-bit significand) before rounding
    once, under the C runtime's default precision control, which Yxy never
    changes. x86-64 and AArch64 need no floor: their baselines have scalar
    IEEE arithmetic of both widths.
13. **The floating-point environment** ([EFF-5], [FLT-3]). Yxy code runs with
    round-to-nearest, no trapped exception, and subnormals kept; the runtime
    never changes this. Foreign code must leave the environment as it found
    it before it returns to Yxy code or calls into it — a precondition of the
    `ffi` boundary, like not unwinding across Yxy frames. A `--link` input
    must not define `fma` or `fmaf`, which the generated code calls on the
    x86 targets.
14. **The C boundary** ([ABI-2]). `f32` crosses as C's `float` and `f64` as
    C's `double`, in `extern` and `export` functions, on every target.
    `fmaf` joins the C library functions that no `export fn` may be named
    ([ABI-3] (b)); `fma` is a prelude name, so neither an `extern` nor an
    `export` function can take it.
15. **Printing** is not in this version: `Console` has no float operation
    (`OPEN.md` #8). A program prints an integer conversion.

### Names now reserved

`f32`, `f64`, `fma`, `to_float`, `truncate_to_int` and
`checked_truncate_to_int` join the prelude names ([PRG-2]). A program that
used one of them as the name of an item, a parameter, a local variable or an
import was valid before this decision and is refused now (the compiler's
E0205, with a note that names this decision); as with decision 0015 no
mechanical fix is offered, since a rename changes every use. Recorded in
`OPEN.md` #48. Struct fields are not names in scope ([STRUCT-1]): a field
named `f32` stays valid.

### Evidence (the compiler's `docs/implementation/STATUS.md`, R900–R915)

A corpus of `f32` and `f64` operations, conversions, traps and values runs in
the three engines (native `-O0`, native `-O2`, the reference evaluator) on
every target the suite runs, compared bit for bit — NaN by class — with
values derived independently of the compiler (exact rational arithmetic and
the rounding written from IEEE 754); the generated code is checked to carry
no fast-math flag, to keep `a * b + c` as two instructions at `-O2` where the
CPU has `fmadd`, and to use SSE2 on i686.

## Alternatives

- **The other options of T16.** (1) Strict with no `fma`: gives up fused
  multiply-add, which numeric kernels need for accuracy and speed, even where
  the code asks for it. (3) Contraction by default, as C's
  `-ffp-contract=on`: the rounding depends on the CPU (the three
  instructions of Context for one expression), so results differ between
  targets and the evaluator, which is pure IEEE, disagrees with native code;
  it contradicts [TGT-1], under which the target does not change a
  program's meaning. (4) Fast-math behind a build option: results would
  depend on the build and on the optimizer's choices, against the master
  prompt (release builds do not change meaning); not even as an opt-in in
  this version. Chosen: (2).
- **A default float type** (`f64`, as in Rust, Go and C). A hidden default
  decides a program's precision silently (M-2), and the integers have none
  ([TY-4]). A default can be added later without breaking programs; removing
  one could not.
- **Type suffixes** (`1.5f32`). [LEX-9] refuses them for integers; context
  gives the type. Additive later.
- **Other spellings** (`1.`, `.5`, `1e5`, `1.5E3`, `1.5e+3`). Each is a
  second spelling of a value a reader must decode, and `yxy fmt` keeps a
  literal's spelling; one spelling, with mechanical fixes for the others,
  keeps sources uniform. **Hexadecimal floats** (`0x1.8p3`) give exact bits
  and would help numeric libraries and tests; deferred, additive.
- **Integer literals as floats when exact** (Zig's comptime integers,
  Swift's literal protocols). Shorter, but `x: f64 := 1` then reads as an
  integer that became a float; no conversion is implicit in Yxy. The error
  has a mechanical fix; accepting it later would only accept more programs.
- **Float to integer that saturates** (Rust's `as` since 1.45: NaN gives 0,
  out-of-range values clamp) changes a value silently (M-2); **undefined
  behaviour** (C) is excluded by [TRAP-3]; **only an `Option`** makes every
  conversion verbose where an out-of-range value is a bug. Chosen: a trapping
  conversion, as integer arithmetic traps when its result is not
  representable ([NUM-1]), and its checked form, as `checked_*` does for the
  operations of §6.4 (M-4: the trapping form is documented as trapping, and
  a recoverable one exists). Rounding modes other than toward zero
  (`round`, `floor`) belong to the math functions, deferred.
- **A new trap kind or `T0001`**. "Integer overflow" (`T0001`) names an
  arithmetic result out of range; a NaN converted to an integer is not an
  overflow, and tools that filter by kind would count it wrongly. A new kind
  keeps each kind one class of failure; the kinds of the trap event are an
  open enumeration (the compiler's implementation decision 0010, amended).
- **Conversion surface.** `as` casts: reserved, and [CONV-3] refuses
  truncating casts; methods (`x.to_f64()`): there are no method calls
  ([GR-1]); type names as functions (`f64(x)`): no type is callable; an
  extended `widen` for exact conversions (small integers to floats, `f32` to
  `f64`): `to_float` covers them, and a statically exact form can be added
  later.
- **`fma` surface.** A method (`a.mul_add(b, c)`, Rust): no method calls; an
  operator: none of the mainstream languages has one; `fma(a, b, c)`, the
  name of IEEE 754 and C, as a prelude function like `checked_add`: chosen.
  **Implementation where the CPU has no instruction**: a software fused
  multiply-add inside each module (no C symbol, needed by the freestanding
  profile, `OPEN.md` #41) — deferred, since a correct one for `f64` is long
  (128-bit intermediate) and must be proved; a higher CPU floor (x86-64-v3
  and an FMA i686) — excludes older CPUs for every program, not only those
  that use `fma`; refusing `fma` on those targets — a program's validity
  would depend on the target, against [TGT-2].
- **`%` as C's `fmod`** (`frem` in LLVM). No target has an instruction, so it
  is a call of the C library (another interposable symbol and `-lm`), and
  the remainder has two definitions (truncated `fmod`, IEEE 754
  `remainder`). Deferred with the math functions.
- **Float literal patterns** with IEEE equality (Rust allows them, with
  lints). A NaN arm never matches, `0.0` matches `-0.0`, and coverage can
  only be proved with `_`; bitwise matching would surprise more. Refusing is
  reversible.
- **NaN canonicalized** after every operation (so its bits are specified).
  One more instruction per operation, for bits that nothing observes in this
  version; **a specified payload propagation** cannot be promised without
  it, since targets differ (Context). Chosen: sign and payload unspecified;
  an operation that exposes bits later must canonicalize or say so.
- **i686.** x87 with its precision control set to 53 bits: still wider
  exponents (different overflow and underflow) and double rounding of
  subnormals, so not bit-identical; clang's default CPU: depends on how the
  C compiler was built (some distributions default i686 to CPUs without
  SSE2); refusing floats on i686: it is a code-generating target that the CI
  runs. The SSE2 floor (`pentium4`, 2000) is fixed by the compiler in the
  code it generates, whatever clang's default.
- **Printing floats** now, as shortest round-trip (Rust, JavaScript) or with
  17 significant digits. Either needs a correctly rounded binary-to-decimal
  conversion in the runtime (which uses `write` alone, [CON-4]) and the same
  in the evaluator, with the text specified exactly (exponent form, `-0`,
  infinities, NaN); deferred with text formatting (`OPEN.md` #7, #8).
- **More types** (`f16`, `f128`, x87's 80-bit): no need yet; `f16` has no
  scalar arithmetic on baseline x86-64 and `f128` is a library everywhere.
  Refused with the compiler's E0900.
- **Other rounding modes, exception flags, `strictfp`**. Not in this
  version: nothing can observe them, and LLVM's default floating-point
  environment is exactly the policy above.

## What could change it

- An interface for reorderable reductions, after the *as-if* reading
  (`OPEN.md` #43), with the internal benchmark B2 of the compiler repository
  (dot product and sum of 10^7 floats, strict, with `fma` and reassociated,
  C as the ceiling).
- Access to the bits of a float (`to_bits`, `from_bits`), which must
  canonicalize NaN or document its bits; printing floats; hexadecimal
  literals.
- The math functions (`sqrt`, `abs`, `floor`, `round`, `min`, `max`,
  `copysign`, a NaN test) and constants (infinity, NaN, the largest value,
  epsilon).
- The freestanding profile (`OPEN.md` #41), which has no C library for `fma`.
- Evidence that the SSE2 floor excludes a real i686 user.
- The author's review of the experimental decisions (A2).
