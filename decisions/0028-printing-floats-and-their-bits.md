# Decision 0028: Printing floats and their bits

- Status: Experimental. It carries out the author's answer (b) to L-0019 Q1
  at the gate of 0.1 (the review request of 2026-09-29 in the control
  repository, `reviews/2026-09-29-gate-0.1-request.md`): bring forward,
  before the opening, only the printing of floats — the shortest text that
  reads back as the same value, `-0`, `inf`, a canonical NaN — together
  with `to_bits`. Every choice inside that answer (the layout of the text,
  where scientific notation starts, the spelling of NaN, the names, and
  `from_bits`) is the orchestrator's, with the alternatives below, for the
  author's review; its state is controlled in the register of the control
  repository.
- Date: 2026-10-05
- Origin: decision 0019, item 15 and question Q1; `OPEN.md` #8; the gate
  of 0.1, L-0019 Q1.
- Spec: `semantics.md` [CON-2] (amended), §6.6 ([FLT-9] amended, [FLT-11]
  and [FLT-12] new), [CONST-7] (the note on conversions), §12; decision
  0019 (item 15 and Q1); `OPEN.md` #8 and #48.
- Implementation: the compiler's `docs/implementation/STATUS.md`,
  R1440–R1445 (R915 changed).

## Context

Decision 0019 gave the language `f32` and `f64` but no way to show one:
`Console` had no float operation (item 15), a program printed
`truncate_to_int` of its floats, and the compiler's own corpus read float
bits through C hooks. Q1 of that decision kept it so until the work on
text formatting (`OPEN.md` #7), with the alternative of bringing forward
only the printing; the author chose the alternative, before the opening,
since without it floats serve little to whoever evaluates the language.

Two things had to be fixed exactly for the text to be the same in every
engine and on every target ([FLT-3]): **which decimal** stands for a value,
and **how it is laid out**. Printing NaN, and the bits of a float, meet
[FLT-9], under which the sign and the payload of a NaN are not specified
(they differ between targets and between a value computed when compiling
and one computed when running): no operation may show them.

## Decision

1. **Operations** ([CON-2]). `console.print_f32(value: f32)` and
   `console.print_f64(value: f64)` write the text of the value ([FLT-11]),
   with no line break, and return `()`. They are operations of `Console`
   like `print_u64` and `print_i64`: they perform `console` ([CON-3]),
   write all their bytes with one write before they return and trap with
   *console write failed* when the stream refuses it ([CON-4]), also on a
   console in memory ([CON-5]). Each takes its own type: an `f32` is
   printed as an `f32`, never widened to `f64`, and an integer or a float
   of the other type is a type error (no implicit conversion, [FLT-7]).

2. **The decimal** ([FLT-11]). For a finite value `x` of the float type `T`
   that is not zero, the decimal printed is the one, among the decimals
   `d·10^k` (`d` and `k` integers) that read back as `|x|` — rounded to
   nearest, ties to even, in `T`, as a float literal is ([FLT-2]) — with the
   fewest significant digits; among those, the closest to `|x|`; of two
   equally close, the one whose last digit is even. (Its significant
   digits `d1 d2 … dn`: `n >= 1`, the first and the last not zero.) At most
   9 digits for `f32` and 17 for `f64`.

3. **The layout** ([FLT-11]). Let `e` be the exponent of the first digit:
   the decimal is `d1.d2…dn × 10^e`.
   - A **negative** value (sign bit set: `-0.0` and `-inf` included) starts
     with `-`. Nothing else has a sign.
   - When `-4 <= e <= 15`, **plain**: for `e >= 0`, the first `e + 1`
     digits (with zeros after the digits when `n <= e`), `.`, and the
     digits left, or `0` when none are; for `e < 0`, `0.`, then `-e - 1`
     zeros, then the digits.
   - Otherwise **scientific**: `d1`, `.`, the digits `d2…dn` or `0` when
     `n = 1`, `e`, and the exponent in decimal, with `-` when it is
     negative, no `+` and no leading zeros.
   - **Zero** is `0.0` or `-0.0`; the **infinities** are `inf` and `-inf`;
     every **NaN** is `nan`, whatever its sign, payload and quietness
     ([FLT-9]).

   So every finite text has a `.`, and is a float literal ([LEX-19]),
   negated by its `-` ([FLT-2]), whose value in `T` is `x`: the text reads
   back as the same value, also in Yxy. The text does not depend on the
   target, the engine, the optimization level, a locale, or the C library.
   It is at most 24 bytes for `f64` and 19 for `f32`.

   | Value | Text |
   |---|---|
   | `1.5`, `0.1`, `100.0` | `1.5`, `0.1`, `100.0` |
   | `0.1 + 0.2` (`f64`) | `0.30000000000000004` |
   | `0.1 + 0.2` (`f32`) | `0.3` |
   | `0.1` (`f32`) widened to `f64` by `to_float` | `0.10000000149011612` |
   | `1.0e15`, `1.0e16`, `1.5e16` | `1000000000000000.0`, `1.0e16`, `1.5e16` |
   | `0.0001`, `0.00012`, `0.00001` | `0.0001`, `0.00012`, `1.0e-5` |
   | the smallest `f64`, 2^-1074 | `5.0e-324` |
   | the largest `f64` | `1.7976931348623157e308` |
   | the smallest and the largest `f32` | `1.0e-45`, `3.4028235e38` |
   | `9007199254740993.0` (`f64`: 2^53) | `9007199254740992.0` |
   | `562949953421312.25` (`f64`, a tie of `….2` and `….3`) | `562949953421312.2` |
   | `1.0e23` (`f64`: 99999999999999991611392) | `1.0e23` |
   | `-0.0`, `1.0 / 0.0`, `-1.0 / 0.0`, `0.0 / 0.0` | `-0.0`, `inf`, `-inf`, `nan` |

4. **Bits** ([FLT-12]). `to_bits(x)` gives the IEEE 754 encoding of the
   float `x`: a `u32` for an `f32`, a `u64` for an `f64`. Every NaN gives
   the **canonical NaN**: the quiet NaN of sign 0 and payload 0,
   `0x7fc00000` and `0x7ff8000000000000`; every other value its own bits
   (`to_bits(-0.0)` is `0x8000000000000000`). `from_bits(b)` gives the
   float whose encoding is `b`: an `f32` for a `u32`, an `f64` for a `u64`;
   an encoding of a NaN gives the canonical NaN. The float type is the
   operand's (`to_bits`) or the context's, whose bits the operand must be,
   or else the operand's (`from_bits`); with none it is an error, as for a
   float literal ([TY-4]). Neither traps nor performs an effect, and
   neither is part of a constant expression ([CONST-2]). So
   `from_bits(to_bits(x))` is `x`, bit for bit, for every `x` that is not
   NaN, the sign and the payload of a NaN stay unobserved ([FLT-9]), and no
   signaling NaN comes from Yxy code.

5. **Names.** `to_bits` and `from_bits` join the prelude names ([PRG-2]),
   as the operations of §6.6. A program that used one of them as the name
   of an item, a parameter, a local variable or an import, valid before, is
   refused (the compiler's E0205, with a note that names this decision); as
   with decision 0019 no mechanical fix is offered, since a rename changes
   every use. Recorded in `OPEN.md` #48. Struct fields keep any name.

6. **Implementation** (not a rule). The text is defined by items 2 and 3,
   not by an algorithm: any correct one gives it. The compiler uses
   Schubfach (below), with integer arithmetic only, once in its runtime
   (the generated code) and once in its reference evaluator, and checks the
   two against each other and against an independent oracle (R1441,
   R1442).

## Alternatives

- **Fixed precision**: 17 significant digits for `f64` and 9 for `f32`
  (C's `%.17g`), which always reads back. The text shows digits that are
  not the value's (`0.1` is `0.10000000000000001`), and a person reading
  `0.30000000000000004` learns something that `0.30000000000000004441`
  hides. Rejected.
- **C's `%g`** (6 significant digits): does not read back
  (`0.30000000000000004` and `0.3` both print `0.3`), and through `printf`
  the text depends on the C library and the locale (`,` as the decimal
  point), which the runtime does not use ([CON-4]: `write` alone). Rejected,
  as is calling the C library for any of it.
- **The algorithm.** Dragon4 (Steele and White, 1990): exact with big
  integers, slow; Grisu (Loitsch, 2010): fast but fails on about 0.5 % of
  values, which then need Dragon4 (Rust's standard library does this);
  Ryu (Adams, 2018): proven, tables of about 10 KB for `f64`; Dragonbox
  (Jeon, 2020): faster, longer to prove and to write; **Schubfach**
  (Giulietti, 2020–2022, Java's `Double.toString` since JDK 19): proven,
  one table of 617 numbers of 126 bits, a short body with no fallback.
  Chosen: Schubfach, for the size of the code and of the proof. Java's own
  rule is not taken: it never prints one digit where two exist
  (`4.9E-324` for the smallest `f64`, here `5.0e-324`, the shortest).
- **Ties.** Two shortest decimals equally close to the value (`562949953421312.25`)
  happen; Ryu, Schubfach, JavaScript and Python take the even digit, Rust's
  standard library rounds up. Chosen: the even one, the rule of IEEE 754
  reading itself.
- **Where scientific notation starts.** JavaScript writes plain from
  `1e-7` to below `1e21`; Python's `repr` and Rust's `{:?}` from `1e-4` to
  below `1e16`. Chosen the latter, familiar and short: every integer an
  `f64` holds exactly up to 2^53 prints plain, and no plain text has more
  than 16 digits before its `.` or more than three zeros between its `.`
  and its first digit. Always plain (`1.0e300` would be 301 digits) and
  always scientific (`100.0` as `1.0e2`) were rejected. The thresholds are on the decimal's exponent,
  which equals comparing the value with the float nearest `1e-4` and
  `1e16` of its own type.
- **`1e16` and `1e+16`.** The literal grammar ([LEX-19]) writes
  `digits.digits` before `e` and no `+`; printing `1.0e16` keeps every text
  a literal. Likewise a plain integer always ends in `.0`.
- **NaN.** `NaN` (Rust, Java, JavaScript) or `-nan` (glibc's `printf`, the
  sign of the bits): the sign and the payload are not specified ([FLT-9]),
  so they are never shown; `nan`, in lower case like `inf`.
- **Widening `f32`** to `f64` before printing, as C's `printf` does:
  `0.1` of an `f32` would print `0.10000000149011612`. The text of a value
  is the text of its type; a program that wants the `f64` digits converts
  with `to_float` first.
- **Names.** `print_f32` and `print_f64` follow `print_u64` and
  `print_i64`. One `print_float` for both would need overloading, which
  the language does not have; a method on the value (`x.print(console)`),
  method calls ([GR-1]).
- **`to_bits` without canonicalizing**: a program printing
  `to_bits(0.0 / 0.0)` would print different numbers on AArch64 and x86,
  and at `-O0` and `-O2` (decision 0019, Context), against [FLT-3]. A NaN
  **canonicalized after every operation** costs an instruction per
  operation (rejected by decision 0019); canonicalizing in `to_bits` costs
  an `and`, a comparison and a selection in it alone. Chosen.
- **`from_bits` keeping a NaN's payload**, or its quietness: Yxy code could
  then make signaling NaNs, which [FLT-9] leaves to foreign code, and
  values whose printing and comparisons agree but whose bits differ only
  by an unspecified payload. Chosen: the canonical NaN.
- **`from_bits` typed only by context** (as `to_float`): `x := from_bits(b)`
  with `b: u64` would need an annotation; the operand's type decides
  unambiguously (`u32` is `f32`'s, `u64` is `f64`'s), so both are taken.
- **The routine in Yxy**, in a package of the standard library, so that the
  evaluator would run the same code as the generated one: the language has
  no constant arrays ([CONST-1]) for the table, no 128-bit product, and a
  console operation is not a library function. The compiler instead writes
  the algorithm twice, in its runtime and in its evaluator, and tests them
  against each other (every `f32`, millions of `f64`).
- **Waiting for text formatting** (`OPEN.md` #7), the choice of decision
  0019: the author chose to bring the printing forward.

## What could change it

- Formatting into text (`OPEN.md` #7): a precision, a width, a form chosen
  by the program, printing into a `String`; additive.
- Reading floats from text; hexadecimal float text, with hexadecimal
  literals (decision 0019).
- The author's review: the thresholds of scientific notation, the spelling
  of NaN and of the infinities, the canonical NaN.
- Evidence that a correct shortest conversion disagrees with the text of
  item 2 (the compiler checks every `f32` and millions of `f64` against
  Rust's correctly rounded reading).
