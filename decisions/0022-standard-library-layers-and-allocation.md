# Decision 0022: The layers of the standard library, allocation and out of memory

- Status: Accepted — **experimental**. It carries out two choices of the
  provisional surface the author authorized on 2026-09-29 (R-7 of the
  decision register in the control repository, `plans/decisions/README.md`):
  allocation **(d)**, the layers `core`, `alloc` and `std`, and out of
  memory **O2**, the plain form traps and a `try_` family returns `Result`.
  Reviewed by the author at the gate of 0.2, with decision 0020.
- Date: 2026-09-30
- Origin: R-7; the study of TASK-20260926-040
  (`plans/evidence/TASK-20260926-040/options.md` §9 and §10, and
  `deliverable.md`); decision 0020, part 1 (the rows "Allocation" and
  "Running out of memory"); task TASK-20260926-044 and its gates in the
  master plan (`plans/reviews/2026-09-26-master-plan.md`).
- Spec: `modules.md` §4.2 ([STD-1]–[STD-8]), [PKG-7]; `semantics.md` §7
  (table), §7.2 ([ALLOC-1]–[ALLOC-4]), [TRAP-1]; `OPEN.md` #30 and #41.
- Amendment (2026-10-04, experimental): decision 0026 adds the packages
  `std/fs`, `std/socket`, `std/time`, `std/rand` and `std/capture` ([STD-8])
  and the root capabilities `Files`, `Net`, `Clock` and `Random`, of the
  `std` layer as `Console` is ([STD-5]); it decides with the measure of
  part 6 (decision 0026, part 10), which the compiler's test takes again
  over the corpus with its programs. Decision 0030 (2026-10-06,
  experimental) adds the packages `std/process` and `std/input` and the
  root capabilities `Args`, `Env`, `Stdin` and `Stderr`, also of the `std`
  layer ([STD-5], [STD-8]).
- Amendment (2026-10-06, experimental): decision 0029 implements the
  `alloc/list` and `alloc/boxed` of part 5 and decides the error of a
  `try_` operation that takes a value, `alloc.Refused<T>` (the value given
  back with the `AllocError`), which [ALLOC-3] now names.
- Author requirements it follows (register of the control repository):
  M-2 (nothing silent), M-4 (no abort hidden behind an API presented as
  recoverable), E-2 (effects checked transitively), E-3 (a capability is not
  an effect, and no capability is a global singleton).

## Context

`modules.md` reserved the roots `core`, `alloc` and `std` for a standard
library supplied by the toolchain ([STD-1]) and layered it ([STD-2]):
`core` needs neither allocation nor an operating system, `alloc` adds what
needs an allocator, and the hosted `std` may use both. The library did not
exist, so every standard import was an error ([STD-4]). Decision 0015 gave
the language its first capability, the console, as a prelude type
(`Console`) with three operations, and asked that the redundancy of a
`Console` parameter and `effects { console }` be measured before the
catalogue of capabilities (TASK-20260926-048).

The study of TASK-20260926-040 compared three ways of saying where memory
is allocated — (a) an effect `alloc`, (b) the allocator as a value, (d) the
layers only — and three ways of running out of memory — O1 every allocation
returns a `Result`, O2 two families, O3 trap only. R-7 chose (d) and O2. On
the study's eight programs that allocate, (a) put the effect on 16
functions, (b1) an allocator parameter on as many, and (d) nothing on any
function: 8 packages import `alloc/…` directly and none through another.
What (d) cannot say is that one function of a package that imports `alloc`
does not allocate (the study's A11: its handler cannot say `effects {}`
means "no allocation"). O2 keeps the plain operation short and gives the
recoverable form a name of its own, so that no API presented as recoverable
hides an abort (M-4).

What the language has today allocates nothing: there is no owned string, no
list and no box. Their operations need destructors (decision 0021),
`inout` (decision 0020) and the owned text of TASK-20260926-043. This decision therefore makes the layers
real and importable, decides how a program gets its layer, and fixes the
policy of allocation and of running out of memory that those types will
follow, with the place and the shape of their operations; the types
themselves come with those tasks.

## Decision

### 1. The standard library exists, with one package per layer ([STD-4], [STD-8])

The standard library is supplied by the toolchain at its version, as
[STD-1] says, and an import of one of its packages works in a module and in
a standalone file ([PKG-7]). An import of a standard path that names no
package of it is an error (the compiler's E0804, which lists the packages it
has). In this version:

| Package | Layer | What it has |
|---|---|---|
| `core/ascii` | `core` | Classification of ASCII bytes and their case: `is_digit`, `is_upper`, `is_lower`, `is_letter`, `is_alphanumeric`, `is_space`, `to_upper`, `to_lower`, `digit_value` (`Option<u8>`). Bytes, not characters (M-3): a byte outside ASCII is none of these and keeps its case |
| `alloc` | `alloc` | `AllocError` (`CapacityOverflow`, `OutOfMemory`), the error of every `try_` operation ([ALLOC-3]), and `describe(e) -> &str`. Nothing of it allocates |
| `std/print` | `std` | Writing through a `Console` the caller passes: `line(console, text)`, `newline(console)`, `boolean(console, value)`, `line_u64(console, value)`, `line_i64(console, value)`. Each traps as the console does ([CON-4]) and says so |

`Console` stays a prelude name ([PRG-2], decision 0015) and belongs to
`std`: a package that names it is of the `std` layer ([STD-5]). No
expression creates a `Console`: the hosted runtime gives it to `main`, and
every function of `std/print` receives it as a parameter (E-3). The
package is called `print`, not `console`, because a package's local name
cannot be the name of a parameter of the file that imports it ([IMP-5]),
and `console` is the name every printing function of the corpus gives its
capability.

The library's packages follow [STD-2]: a package of `core` may import only
`core`, one of `alloc` only `core` and `alloc`, and one of `std` any layer.
Today none of the three imports anything. The compiler checks the rule
over its sources, so a new import is checked, and that every source is in
the canonical form of `yxy fmt`.

The library's code is compiled into the executables of the programs that
import it; the reach of the project's license over that code is pending the
author's A11 in the decision register of the control repository.

### 2. A program gets its layer from what it uses ([STD-5]–[STD-7])

Nothing declares a layer. The layer of a package of the standard library is
the root of its path; the layer of another package is the highest layer of
the packages it imports, directly or not, and `std` when one of its
functions has a `Console` parameter or variable; otherwise `core`
([STD-5]). The layer of a program is the highest layer of its packages
([STD-6]). The tools report both (`yxy inspect --json`: `layer`, and the
`layer` of each package). The layer is read with `declares_foreign`: a
program of the `core` layer that declares foreign functions may still
allocate through them (below).

A layer says what the Yxy code of the package may need ([STD-7]): a `core`
package allocates nothing and uses no operating-system service of the
library; an `alloc` package may allocate through the allocator the program
supplies (the hosted runtime's, [ALLOC-1]); a `std` package may use the
hosted runtime's capabilities. Code reached through `ffi` is outside the
layer: foreign code may allocate or call the operating system whatever the
layer of the package that declares it, as `ffi` is outside the other
guarantees ([EFF-5]). The trap report of a hosted program ([TRAP-1]) is the
hosted runtime's in every layer; a freestanding profile gives a `core`
program its own (OPEN #41).

The import is the declaration: under (d), a package that imports `alloc/…`
may allocate anywhere, so the imports already say what a declared layer
would. A declared ceiling, refused when a package uses more, is left to the
freestanding profiles of [STD-2] (OPEN #41), where the build must refuse a
program that needs an operating system; adding it later accepts every
program valid today that stays under the ceiling it declares.

### 3. Allocation is what the `alloc` layer gives ([ALLOC-1], [ALLOC-2])

- Only the operations of the packages of `alloc` (and those of `std` that use
  them) allocate, and the explicit copy of an owned heap value, which is
  deep ([OWN-6]). Allocation is not an effect: `effects {}` keeps meaning "no
  tracked effect" ([EFF-2]), not "does not allocate"; and no operation takes
  an allocator argument.
- "Does not allocate" is said of a package or of a program: its layer is
  `core`. It cannot be said of one function of a package of a higher layer
  (the cost of (d) that the study measured).
- The hosted runtime supplies the allocator. A freestanding program that
  imports `alloc` supplies one; how is part of its profile (OPEN #41).
- Freeing never fails. An owned heap value is freed when it is destroyed
  (destructors: decision 0021).

In this version no operation allocates: the package `alloc` holds only its
error. The compiler's test is the gate of TASK-20260926-044: a `core`
executable without foreign code (no `extern fn`, no file given with
`--link` or `--lib`) links with no allocation function of the C library,
the C++ runtime or the system (`nm -u`), nor do the `alloc` and `std`
programs of the test, and their objects for every code-generating target
refer to none; the same check finds `malloc` when a linked C file calls
it. Foreign code is outside the layer (part 2), so the gate says nothing of
a `core` program that declares foreign functions.

### 4. Out of memory: two families, O2 ([ALLOC-3], [ALLOC-4])

- The **plain** form of an operation that allocates (`string.from`,
  `string.push`, `list.push`, `boxed.new` below) traps when the allocator
  has no memory or the size asked for does not fit in `usize` or in the
  target's address space: the trap kind *allocation failed*, exit status
  101, reported at the operation ([TRAP-1], [TRAP-3]; the compiler reserves
  the code T0007). Its documentation says that it traps and with which
  code, as the console's operations do for T0005 ([CON-4]).
- The **`try_`** form (`string.try_from`, `string.try_push`, `list.try_push`,
  `boxed.try_new`) returns `Result<T, alloc.AllocError>`: `Err(OutOfMemory)`
  when the allocator has no memory, `Err(CapacityOverflow)` when the size
  does not fit. It never traps for lack of memory, and on failure it leaves
  its `inout` operands as they were.
- `.copy()` of an owned heap value allocates and traps like a plain form;
  so does `[v; N]` of such a value, whose N − 1 copies allocate (the known
  gap of the study under O1 does not arise under O2). A recoverable copy
  (`try_copy`) is not part of O2; adding it later is additive.
- M-4, as a rule on the library ([ALLOC-4]): every public function of `alloc`
  and `std` that can trap says so, with the code of each trap it can reach;
  a function that cannot trap does not say that it traps; and a function
  presented as recoverable — every `try_` function, every function whose
  result is a `Result` or `Option` for a failure — never traps for lack of
  memory. The compiler's test derives, from the facts of a program that
  calls every public function of the library, the traps each can reach
  (its checks, the console's T0005, and those of what it calls) and checks
  them against its comment, and checks that a `try_` function returns
  `Result<…, alloc.AllocError>`.
- In this version no operation allocates, so the part of M-4 about running
  out of memory is met only vacuously: no function can reach T0007 and the
  library has no `try_` function. That part moves to TASK-20260926-043,
  whose owned string brings the first operations that allocate, with a test
  of the trap and of the `Result`. The compiler reserves T0007 in a table
  its tests check, so that no other trap takes the code.

### 5. Where the owned heap types go (declared, not implemented)

| Package | Type | Plain form (traps, T0007) | `try_` form (`Result<…, alloc.AllocError>`) | Waits for |
|---|---|---|---|---|
| `alloc/string` | `String`, owned UTF-8 text (M-3) | `string.from(t: &str) -> String`; `string.push(inout s: String, t: &str)`; `string.with_capacity(n: usize) -> String` | `string.try_from(t: &str) -> Result<String, AllocError>`; `string.try_push(inout s: String, t: &str) -> Result<(), AllocError>`; `string.try_with_capacity(n: usize) -> Result<String, AllocError>` | destructors (decision 0021) and `inout` (decision 0020), the owned text (TASK-20260926-043) |
| `alloc/list` | `List<T>`, a growable sequence | `list.new() -> List<T>` (allocates nothing); `list.with_capacity(n: usize) -> List<T>`; `list.push(inout l: List<T>, take v: T)` | `list.try_with_capacity(n: usize)`; `list.try_push(inout l: List<T>, take v: T)`, whose error gives `v` back with the `AllocError` | the same, and a generic type of the library (OPEN #5) |
| `alloc/boxed` | `Box<T>`, one owned value on the heap | `boxed.new(take v: T) -> Box<T>` | `boxed.try_new(take v: T)`, whose error gives `v` back with the `AllocError` | the same |

The exact type of the error of a `try_` operation that takes a value (it
must give the value back, or the failure would destroy it) is decided with
`List` and `Box`. A view of an owned string is a `&str` ([TEXT-1]) whose
duration the rules of TASK-20260926-042 bound. `std` uses these types for
what needs them (formatting into text, reading lines); until then `std` has
only what writes through the console.

### 6. Measure of the redundancy of direction (c), for TASK-20260926-048

Decision 0015 measured, over the programs of its change, that the eight
functions that write to the console have both a `Console` parameter and
`effects { console }`. Measured again here, over every valid program of the
compiler's corpus — the examples, the run, pass, ABI and formatter
programs, the benchmarks, the runnable module cases — and the standard
library, each function counted once (the compiler's test
`console_redundancy_of_direction_c_over_the_corpus`, 2026-10-04):

| Measure | Count |
|---|---|
| Valid programs | 95 |
| Functions | 530 |
| With a `Console` parameter | 18 |
| Declaring `console` | 18 |
| Both | 18 |
| A `Console` parameter without `console` | 0 |
| `console` without a `Console` parameter | 0 |
| Writing to the console themselves | 15 |
| Passing their console to a function that takes one | 6 (3 of them without writing themselves) |
| Declaring `console` with another effect (`ffi`) | 2 |
| `main` taking the console | 8 |

On 2026-09-30, before the programs of `inout`, views and destructors
(decisions 0020 and 0021) joined the corpus, the measure had 89 programs
and 470 functions, and every other count was the same.

Of the 18 functions with a `Console` parameter, 10 come from the corpus as
it stood before this decision, and 8 from code written for it, already in
the convention: the 5 functions of `std/print`, the `main` of the test
program of `layers.rs`, of `tests/run/std_layers.yxy` and of the module
case `std_layers`. The 10 of the earlier corpus also hold the pair both
ways. The test pins the two zeros of the table (a `Console` parameter
without `console`, and `console` without a `Console` parameter) and the 8
functions written for this decision, as measured on 2026-10-04. The
language allows both cases, so the zeros are a property of the corpus: a
program that adds one fails the test until the measure is taken again.

In this corpus the pair still carries one bit: a function holds the
capability if and only if it declares the effect, so every one of the 18
`console` words (16 of them the whole `effects { console }` clause) repeats
what the parameter says. What the effect adds is what 0015 listed: a bound
read from the signature that stays transitive when a capability can be
stored or captured, and the rule that already checks `ffi`. The 3
functions that only pass the console on show the transitive part: they
declare `console` because a function they call writes ([EFF-3]), which the
parameter alone would also show only while a capability cannot be stored.
This decision changes neither; the catalogue of TASK-20260926-048 decides
with this measure.

## Alternatives

- **(a) An effect `alloc`** and **(b) the allocator as a value**: not chosen
  by R-7. (a) says "does not allocate" of one function (the study's A11)
  at the cost of the effect on every function that allocates or calls one
  (16 in the study's programs); (b) makes the allocator explicit, arenas
  included, with an argument on as many functions under (b1), or only on
  constructors under (b2), where a function that receives a container can
  still allocate. **(c)**, the allocator as a type parameter, needs user
  generics (OPEN #5).
- **O1, every allocation returns a `Result`**: every function that
  allocates changes its result, or nests it (`Result<Result<T, E>,
  AllocError>`), and `[v; N]` of an owned heap value has no failure path.
  **O3, trap only**: a program cannot recover from running out of memory at
  all.
- **A layer declared by the program** (a key of `yxy.toml`, a clause in the
  package, or a build option such as a profile): a second statement of what
  the imports say; useful as a ceiling that refuses, which the freestanding
  profiles need (above).
- **`Console` moved into a package of `std`** (`import "std/console"`, then
  `console.Console`): it would invalidate every program that prints (OPEN
  #48), and the package's local name would collide with the parameter
  every such program calls `console` ([IMP-5]). The prelude type keeps the
  capability's type one that no program can redefine (decision 0015).
- **The writing package named `std/console`, imported with an alias**
  (`import "std/console" as out`, then `out.line(console, …)`): an alias
  avoids the collision of [IMP-5], and the same alias works for
  `std/print`, so the name `print` only moves the collision (a parameter
  named `print` beside `import "std/print"` is E0202), and
  `print.line(console, …)` beside `console.print(…)` uses `print` in two
  roles. `std/print` was kept because a program that writes needs no alias
  under the name every program of the corpus gives its console; the name
  can still change before the library is stable.
- **One package per layer** (`import "std"`, `import "core"`): a package
  grows with everything its layer holds, and a program imports all of it
  to use a part. Paths under each root, as other packages have, keep
  imports precise; `alloc` is a package itself because its error type is
  shared by every package of the layer (as `options.md` writes it,
  `alloc.AllocError`).
- **The library installed as files beside the toolchain** instead of built
  into the compiler: a second artefact to install and keep in step; the
  compiler's implementation decides it (it builds the sources into the
  compiler and names their files `<toolchain>/<import path>/<file>`).

## What could change it

- The author's review at the gate of 0.2, with decision 0020.
- Decision 0021 (destructors), `inout` (decision 0020) and the owned string
  of TASK-20260926-043 implement the table of part 5 and may refine the shape of the `try_`
  operations that take a value.
- The freestanding profiles (OPEN #41): a declared ceiling, a trap hook,
  and how a freestanding program supplies its allocator.
- The catalogue of TASK-20260926-048, with the measure of part 6: which
  capabilities exist, whether the effect of a capability stays written, and
  how `main` receives them (OPEN #9).
