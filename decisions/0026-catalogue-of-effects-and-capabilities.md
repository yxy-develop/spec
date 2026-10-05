# Decision 0026: The catalogue of effects and capabilities

- Status: Experimental — not accepted. Written under TASK-20260926-048 of
  the master plan (Phase 8), for the author's review. It closes OPEN #9.
  The direction (an effect is the kind of an action, a capability is the
  value that reaches a resource, both checked, no global singleton) is the
  author's requirement E-3; every choice below inside that direction is the
  orchestrator's, with the alternatives that follow.
- Date: 2026-10-04
- Origin: `decisions/OPEN.md` #9; the master plan's row of
  TASK-20260926-048 and its "Portões objetivos (errata)", §5, items 1–6 and,
  after the answers of 2026-09-26, items 7–9; the architecture audit of
  2026-09-26, §12b.1 (the criterion of 12b.1.2, the table of 12b.1.3, the
  conflicts C1 and C4, task 5 (a)–(f)); the measures of the redundancy of
  direction (c) in decisions 0015 and 0022 (part 6).
- Spec: `semantics.md` §7 ([EFF-1], [EFF-2], the table), §7.1 ([CON-1],
  [CON-3], new [CON-5]), new §7.5 ([CAP-1]–[CAP-7]) and §7.6 ([FS-1]–[FS-3],
  [CLOCK-1], [RAND-1], [SOCK-1], [SOCK-2]), [ABI-3] (a), [ABI-4], §10
  ([MAIN-1]), §12; `modules.md` [INIT-1], [STD-5], [STD-7], [STD-8];
  `OPEN.md` #9 (closed), #48, #54–#56.
- Author requirements it follows (register of the control repository):
  E-1 (a small, extensible effect system; `effects {}` is the absence of the
  tracked effects), E-2 (effects checked transitively through calls,
  recursion, callbacks and destructors), E-3 (a capability is not an
  effect; console, files, network, clock and randomness through explicit
  APIs, without global singletons), E-4 (with decision 0024), Y-9 (HTTP,
  ORM and AI are not part of the core), M-2 (nothing silent), M-4 (no abort
  hidden behind an API presented as recoverable), P-2 (traceable costs).

## Context

Before this decision the language tracked two effects, `ffi` and `console`
([EFF-2]), had one capability that the runtime gives, `Console`
([CON-1]), and one created at an unsafe boundary, `Mmio` ([MMIO-1]).
[MAIN-1] called the arrival of the other capabilities in `main` "future
work", and OPEN #9 listed files, network, clock and randomness as
candidates. Any package could still reach those resources with ambient
authority, an `extern fn` and `ffi`.

Decision 0015 chose direction (c) of the audit's §12b.1 C1 for the console:
two layers, the effect (the kind of action, checked by [EFF-3]) and the
capability (the value born in `main` that names the resource). It left the
catalogue, with a measure of the redundancy of the pair, to this task.
Decision 0022 measured it again over the whole corpus (part 6): 18 of 18
functions with a `Console` parameter declare `console`, and the converse;
the pair carried one bit while no capability could be stored. Decision 0025
added function values and generics, so E-2 now has four paths (calls,
recursion, callbacks, destructors), each with refusals.

The audit's criterion (§12b.1.2, recorded as a note in OPEN #9) classifies
a concern by four questions, in order: (1) a value the caller receives is a
**type**; (2) a predicate that must hold at a point is a **contract**; (3)
something that may happen during a call, which a caller must be able to
exclude from the signature alone, is an **effect**, named after the kind of
action, never the resource, and only when the language or the hosted
runtime mediates it; (4) when two calls with the same effect may reach
different resources and the difference matters, a **capability**: a value
obtained from `main` or from another capability. A concern may have facets
in several classes; the table records the primary one.

## Decision

### 1. The classification table (gate item 1)

Every item named by the master prompt (l.220–222 and l.228) is in exactly
one class. Zero items are unclassified.

| Item | Class | Why (the four questions) | Other facets |
|---|---|---|---|
| console | CAPABILITY (`Console`) | Q4: two writes may reach standard output or a console in memory, and tests need the difference ([CON-5]) | EFFECT `console` (Q3); its failure is a trap, T0005 ([CON-4]) |
| files | CAPABILITY (`Files`) | Q4: which files a function reaches matters (tests, isolation) | EFFECT `fs`; TYPE `fs.FsError` |
| network | CAPABILITY (`Net`) | Q4: which network, and which socket (`socket.Socket`, derived) | EFFECT `net`; TYPE `socket.NetError` |
| clock | CAPABILITY (`Clock`) | Q4: a test gives code its own notion of time only through what it receives | EFFECT `clock`; TYPE `time.ClockError` |
| randomness | CAPABILITY (`Random`, the entropy of the operating system) | Q4: which source of entropy | EFFECT `random`; TYPE `rand.RandomError`. A generator with explicit state seeded from it is a TYPE with no effect (none in this version) |
| `ffi` | EFFECT | Q3: foreign code may do anything, and a caller must be able to exclude it from the signature; there is no value to receive (Q4 no: foreign code has ambient authority, which is why it is the unsafe boundary, [EFF-5]) | — |
| allocation | none of the four, because it is a property of the layer: what a package of `alloc` may do (decision 0022, [ALLOC-1]); the noise test made it no effect (16 functions in the study of TASK-040) and R-7 made it no capability (no allocator value) | its failure is a TYPE (`try_`, `Result<…, alloc.AllocError>`) or a documented trap (T0007) |
| typed failure | TYPE | Q1: `Result`, `Option`, `require`, `?`: the caller receives it and handles it | — |
| trap | CONTRACT | Q2: the implicit contract of each primitive operation (overflow, bounds, division); its violation ends the process ([TRAP-1]); not an effect ([EFF-1]) | — |
| local mutation | TYPE | Q1: `mut` and `inout` are in the declaration and the signature ([DECL-2], [OWN-9]); not an effect | — |

### 2. The catalogue (§12b.1, task 5 (a))

Each entry of the catalogue of the hosted resources has its four parts
filled, and passes the four questions (the answers in the last column:
Q1 not a value received, Q2 not a predicate, Q3 may happen in a call and a
caller must exclude it, Q4 the resource reached matters).

| Entry | Effect | Capability value | Failure type | Contract | Q1–Q4 |
|---|---|---|---|---|---|
| console | `console` | `Console`: root, standard output; derived, a console in memory (`capture.text`) | none returned: a write the stream refuses traps (T0005), documented ([CON-4]) | every byte written, in order, unbuffered ([CON-4]) | no, no, yes, yes |
| files | `fs` | `Files`: root, the files under the working directory | `fs.FsError` (`NotFound`, `PermissionDenied`, `IsDirectory`, `InvalidPath`, `TooLarge`, `Other(errno)`) in a `Result` | a path is relative, under the directory, checked before the operating system sees it ([FS-2]); `write` writes every byte or fails; `read` fills from the start or says `TooLarge` ([FS-3]) | no, no, yes, yes |
| network | `net` | `Net`: root, the host's network at the level of sockets; derived, `socket.Socket`, one socket | `socket.NetError` (`PermissionDenied`, `Other(errno)`) | sockets only, IPv4, UDP and TCP (Y-9); a `Socket` is opened close-on-exec ([SOCK-1]) and closed when it is destroyed ([SOCK-2]) | no, no, yes, yes |
| clock | `clock` | `Clock`: root, the clocks of the host | `time.ClockError` (`Unavailable`) | the monotonic clock never goes back; the wall clock is nanoseconds since 1970 UTC, and fails after 2262 (an `i64` of nanoseconds) or, where C's `time_t` has 32 bits (i686 Linux), outside 1970-01-01 to 2038-01-19 03:14:07 UTC ([CLOCK-1]) | no, no, yes, yes |
| randomness | `random` | `Random`: root, the entropy of the operating system | `rand.RandomError` (`Unavailable`) | every byte asked for comes from the operating system's entropy, or the call fails ([RAND-1]) | no, no, yes, yes |

`ffi` is an effect with no capability (above). `Mmio` (decision 0024) is a
capability with no effect: it is created by an unsafe operation, not given
by the runtime, and whether device access becomes an effect stays open
(OPEN #51); it is not a root capability.

No effect of the core has the name of a domain of a library (§12b.1, task 5
(f); C4 (a)): `clock`, `console`, `ffi`, `fs`, `net` and `random` name kinds
of action that the language or the hosted runtime mediates. A domain (a
database, mail, HTTP, inference) is a capability of a library over an
effect of the core (an HTTP client would hold a socket and perform `net`).
Effects declared by libraries are deferred (OPEN #54).

### 3. Effect ≠ capability: two rules, and the matrix (gate item 2)

The **effect rule** is [EFF-3] (the callee's declared effects are a subset
of the caller's, at every call), with [CAP-3] naming the effect each
operation performs. The **capability rule** is [CAP-2] (a root capability is
born only in `main`, travels as a parameter or a local, and no expression
creates one) with [CAP-3] (an operation takes the capability as an
argument). They are distinct rules with distinct ids and distinct
diagnostics.

For one operation of each hosted resource, the matrix (effect declared? ×
capability held?), each cell with its test in the compiler's suite:

| Resource (operation) | effect yes, capability yes | effect no, capability yes | effect yes, capability no | effect no, capability no |
|---|---|---|---|---|
| console (`console.print`; `print.line`) | runs (`run/capability_matrix`) | E0600 (`fail/capability_without_effect`) | E0615 (`fail/effect_without_capability`) | E0615 and E0600 (`fail/capability_matrix_neither`) |
| files (`fs.read`; `fs.write`) | runs: `Err(NotFound)`, `Err(InvalidPath)` (`run/capability_matrix`); a write read back (`compiler/tests/catalogue.rs`) | E0600 | E0615 | E0615 and E0600 |
| network (`socket.udp`) | runs: a socket opens and closes | E0600 | E0615 | E0615 and E0600 |
| clock (`time.monotonic_ns`, `time.wall_ns`) | runs: never goes back | E0600 | E0615 | E0615 and E0600 |
| randomness (`rand.next_u64`) | runs: two values differ | E0600 | E0615 | E0615 and E0600 |

The two mixed cells have their own tests, one file each. A function that
holds a capability but does not declare its effect cannot use it; a
function that declares an effect but holds no capability cannot reach any
resource, and cannot make one (E0611, part 4).

### 4. No global singleton (gate item 3)

- **(a)** [INIT-1] stays: a package has no global variables, so no
  capability is a global value. `static` stays a reserved word ([LEX-6])
  with no meaning; if a later decision gives it one, a static value is never
  a capability nor holds one ([INIT-1], [CAP-7]). Tests:
  `fail/console_global`, `fail/capability_global`.
- **(b)** The hosted library exposes no free function that reaches a hosted
  resource ([CAP-5]; §12b.1, task 5 (d)): every public function of a package
  of `std` that performs an effect of the catalogue receives, as a
  parameter, a capability of that effect — a root capability or one derived
  from it ([CAP-4]). The compiler's test reads the facts of `yxy inspect
  --json` of a program that imports every package of `std` and finds zero
  violations. The operations of the hosted runtime that the library calls
  take the capability too, or a descriptor held by a capability derived
  from it (`socket_close`, called by the destructor of a `socket.Socket`,
  takes the socket's descriptor, and the facts name `std/socket.Socket` as
  its capability), so no code of the library reaches a resource without
  one.
- **(c)** Root capabilities are born only in `main` ([CAP-2], [MAIN-1],
  rewritten without "future work"): the runtime of a hosted program gives
  `main` one value of each root capability its parameters name, and no
  expression of any function creates one (E0611, `fail/capability_conjured`
  for every root capability, `fail/console_conjured` for the console). An
  `export fn` called by C receives no capability: capabilities do not cross
  the C boundary ([ABI-4], E0311, `fail/capability_stored`), so such a
  function reaches no hosted
  resource of the catalogue; what it does through `ffi` is foreign code's
  ambient authority, outside the catalogue ([EFF-5]). A freestanding
  program has no hosted runtime and no root capability ([STD-2]).

### 5. Y-9 (gate item 4)

The catalogue has no HTTP, ORM or AI: the network is reached at the level
of sockets (`std/socket`), and the standard library has no package of a
protocol above them. The compiler's test checks that no effect and no
package path of the library names such a domain (`fail/domain_effects` for
`http`, `db`, `mail` and `ai`).

### 6. Facts (gate item 5; §12b.1, task 5 (e))

`yxy inspect --json` gives, for each function, its declared effects
(`effects.declared`, unchanged) and, new, `capabilities`: each capability
its parameters hold, the parameter itself or a part of it (a field, an
element, the data of a variant), with its path, the capability, whether it
is a root one, and the effect its operations perform. `capability_calls`
lists the operations of the hosted runtime that the library performs, with
the capability each takes and their effects. The
`limits` entry `effects` says what `effects {}` means, in the sentence of
[EFF-1], and that the declared effects are an upper bound of what a
function and everything it calls may perform (§12b.1, task 5 (e)); the
compiler's test checks over the corpus that the effects of every call and
operation of a capability of a function, and those of every destructor its
destructions may run (a destruction names destructors, whose effects are
their own declared ones), are within its declared effects.

### 7. E-1 (gate item 7)

[EFF-1] says, in one sentence that the `limits` of the facts repeat word
for word: "`effects {}` means the absence of the tracked effects: the
function performs none of `clock`, `console`, `ffi`, `fs`, `net` and
`random`, itself or through anything it calls; it does not mean that the
function is pure, that it does not allocate or that it cannot trap". The
catalogue is small and extensible: an effect enters it only by a decision,
and only for a kind of action that the language or the hosted runtime
mediates ([EFF-2]).

### 8. E-2 (gate item 8)

The check is transitive through every path of this version, each with a
`fail/` test of the compiler's suite:

| Path | Tests |
|---|---|
| calls | `effect_violation`, `effect_transitive`, `console_effect_missing`, `capability_without_effect` (the library's functions) |
| recursion | `effect_recursion` (new: mutual recursion and a function that calls itself), `gen_effects_recursion` (through a function value) |
| callbacks and function values | `gen_effects_callback`, `gen_effects_value`, `gen_effects_generic` (a callback parameter of a generic function), `gen_effects_dynamic_drop` |
| destructors | `drop_effects`, `gen_effects_destroy`, `socket_destroyed_without_net` (new: a destructor of the library that performs `net`) |

### 9. E-4 (gate item 9)

Decision 0024 carries it out ([UNS-4]); this decision changes nothing of
it. Its tests stay: `fail/unsafe_safe_function_never_dereferences`,
`fail/unsafe_regions`, and the facts of `unsafe_regions` and
`reaches_unsafe` in the compiler's `unsafe_regions.rs`.

### 10. The redundancy of direction (c): both stay

The measure of decisions 0015 and 0022 (part 6) found that the pair
`console: Console` and `effects { console }` carried one bit while no
capability could be stored or captured. Two facts of this decision make the
effect carry more than the parameter:

- a derived capability of the library may be stored: a `socket.Socket` is
  a struct, which moves and may be a field; its destructor performs `net`
  ([SOCK-2]), so a function that destroys a struct holding one declares
  `net` with no `Net` among its parameters. The parameters no longer show
  every resource a function reaches; the effect still bounds it;
- function values (decision 0025) carry their effects in their type, not a
  capability: `capture.text(console, f)` performs `console` through `f`.

The effect is kept as the transitive bound read from the signature; the
capability as the value that says which resource. The compiler's measure of
decision 0022 (`layers.rs`, `console_redundancy_of_direction_c_over_the_corpus`)
was taken again on 2026-10-04 over the corpus with this decision's programs:
111 valid programs, 690 functions; 41 hold a `Console` and declare
`console`, none holds one without the effect or declares it without one
(13 of them written for this decision); for the other root capabilities,
4 functions hold `Files` and declare `fs`, 5 `Net` and `net`, 4 `Clock` and
`clock`, 4 `Random` and `random`, none holds a root capability without its
effect, and one function declares `net` holding only the derived `Socket`
(the destructor `socket.close`): the case where the parameter no longer
says what the effect bounds. The test pins these counts, derived by hand
from the sources.

### 11. What the compiler implements, and what is deferred

Implemented (the compiler's `STATUS.md`, R1320–R1339): the four root
capabilities, prelude types given to `main`; the effects `fs`, `net`,
`clock` and `random`; the packages `std/fs`, `std/socket`, `std/time`,
`std/rand` and `std/capture`, whose operations reach the C library through
the hosted runtime (its operations are internal to the library, take the
capability, or a descriptor held by a capability derived from it, and
perform its effect, and their C functions are reserved symbols, [ABI-3]
(a); a socket is opened close-on-exec, as a file is); the console in memory ([CON-5]); E0615; the facts;
the tests named above, in the three engines (the reference evaluator calls
the same C functions).

Deferred, each in `OPEN.md`: effects declared by libraries (#54); files
beyond whole reads and writes (handles, directories as capabilities derived
from `Files`, removal, listing, metadata), standard input and standard
error, the arguments and the environment of the process as capabilities
(#55); sockets beyond opening (addresses, binding, connecting, sending,
receiving, IPv6), a clock given to tests (a capability derived from a
function value, as the console in memory is), a seeded generator (#56).

## Alternatives

- **(a) Effects only**, with library functions callable from anywhere
  (§12b.1 C1 (a)): against E-3 (ambient authority), and a test could not
  give a function another console or clock without foreign code.
- **(b) Capabilities only**, the effect deduced from the parameters (C1
  (b)): one word less per signature, which the measure of decision 0022
  found redundant; but a `Socket` stored in a struct, or an effect reached
  through a function value, is then visible only by a data-flow analysis,
  and `effects {}` stops being a transitive bound read from the signature
  (E-2). Rejected for part 10.
- **(d) Effect and capability unified** (the capability in scope is the
  effect, as in Effekt or Scala 3's capture checking): needs closures and
  effect polymorphism (OPEN #53).
- **A coarse effect `io`** for files, network, console: decision 0015
  rejected it for the console; one word for every resource would not let a
  caller exclude the network while allowing files.
- **A directory capability with sub-capabilities now** (WASI's pre-opened
  directories): `Files` is the working directory only; deriving one for a
  sub-directory needs `openat` and a descriptor kept in the capability,
  deferred (#55). Paths are checked by text ([FS-2]), not by the operating
  system: a symbolic link under the directory may lead out of it, as the
  table of §7 says of effects ("not an operating-system sandbox").
- **The capabilities as types of `std` packages** (`fs.Files` instead of a
  prelude `Files`): no new prelude names, so no program that names a type
  `Clock` breaks (OPEN #48); but a root capability has rules the compiler
  enforces (no expression creates one, `main` receives it), and the prelude
  is the closed list of names known to the compiler ([STD-3]); `Console`
  is there already (decision 0022 kept it). Not adopted; the four names are
  recorded in OPEN #48 as a change that invalidates valid programs.
- **A fake console as a separate type** (a `Writer` the tested function
  takes instead of `Console`): the tested function would change, which
  §12b.1 task 5 (c) excludes. **A global switch of the console's stream**
  for tests: a global singleton, against E-3.
- **Every operation as a method on the capability** (`files.write(path,
  data)`, as `console.print`): it needs the error types in the prelude or
  compiler-known operations returning types of a package; functions of the
  package that take the capability keep the prelude to four type names.
- **Failures as traps** (as the console's): a missing file is an ordinary
  outcome; M-4 forbids hiding an abort behind an API presented as
  recoverable, and these are presented as recoverable.

## What could change it

- The author's review: the names of the effects and of the capabilities
  (hard to change once published, §12b.1.7), the prelude names, whether
  `effects { … }` keeps the capability's effect written (part 10), the
  console in memory as the way tests replace the console.
- A measure of the noise of the new effects over a corpus that uses files,
  the network, clocks and randomness (the audit's noise test), which this
  version's corpus is too small for.
- The deferred items of part 11, in particular directory capabilities and
  effects declared by libraries.
