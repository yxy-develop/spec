# Decision 0024: Unsafe regions, the `Mmio` capability, ownership at the C boundary and unwinding

- Status: Experimental — not accepted. Written under TASK-20260926-045 of
  the master plan (Phase 8). Two of its parts carry out choices of the
  author: E-4 (a requirement of the author, register of the control
  repository: "`unsafe` marks manual obligations; a safe function never
  dereferences an arbitrary address just because its body contains
  `unsafe`") and U1 of R-7 (2026-09-29: a foreign unwind aborts at the C
  boundary with a trap report; decision 0020). The route of U1, the
  personality routine (part 5), is the recommendation (a) of item R-7.5 of
  the request for the gate of 0.2 (`plans/reviews/2026-10-04-gate-0.2-request.md`),
  which awaits the author's answer: part 5 depends on that answer, and the
  decision stays experimental at least until it comes. The rest are the
  orchestrator's choices, with the alternatives below, for the author's
  review. Amended the same day by the task's self-check (P45-2, P45-3):
  the event of a trap without a site is `yxy.trap/v2` (part 5), and a value
  read through an `Mmio` handle is not fixed (part 3).
- Date: 2026-10-04
- Origin: TASK-20260926-045; the architecture audit of 2026-09-26, §12b.1
  (C3, option (b); rows 17 and 18; task 7; `plans/reviews/2026-09-26-architecture-audit.md`);
  the study of TASK-20260926-040, D5 (`plans/evidence/TASK-20260926-040/deliverable.md`);
  `decisions/OPEN.md` #34 and #45; the master prompt §8 (l.200-202):
  "do not expose a safe function that accepts an arbitrary integer address
  and dereferences it just because the body contains `unsafe`; require an
  explicit unsafe boundary or a previously validated capability".
- Spec: `semantics.md` §7.3 ([UNS-1]–[UNS-5]), §7.4 ([MMIO-1]–[MMIO-4]),
  [EFF-5] and the table of §7, [ABI-3] (c), [ABI-4], [TRAP-1], [PRG-2],
  §12; `syntax.md` [LEX-5], [LEX-6], [GR-10] and the grammar;
  `decisions/OPEN.md` #9, #34, #45 (closed), #48 and #51.
- Author requirements it follows: E-1 (a small, extensible effect system:
  no new effect), E-2 (transitive verification: the reach of unsafe regions
  follows calls, recursion and destructors), E-3 (a capability is not an
  effect, and no capability is a global singleton), E-4 (above), M-2
  (nothing silent: a foreign unwind gets a report).

## Context

Until now `ffi` was the whole unsafe boundary ([EFF-5]): Yxy had no
pointers, an integer became an address only on the C side, and the
memory-safety guarantees held for code whose effects exclude `ffi`. Two
things were missing. A program that drives hardware had to write its
register accesses in C, and every caller up to `main` declared `ffi`
(the audit's "contagion", §12b); and nothing marked the obligations a
program takes on when it touches memory the compiler cannot reason about.

The audit's §12b.1 C3 compared three treatments of `unsafe`: (a) an effect
propagated through signatures, (b) a contract local to its function with a
fact "reaches unsafe code" computed over the call graph, (c) no `unsafe`,
everything unsafe in C; it recommended (b), with MMIO as a capability, a
typed handle created once at an unsafe boundary (row 17). The author's E-4
adds the requirement that a region never turns a safe function into one
that dereferences whatever integer its caller gives.

Two questions of the C boundary were deferred to this task. OPEN #34 (C
ABI for structs) is where ownership across the boundary was to be decided:
what crossing does to owned values, to `take`, `inout`, destructors and
the views `&str` and `&[T]`. OPEN #45 (unwinding): decision 0020 chose U1,
a report at the boundary, but the compiler still ended the process through
the C++ runtime's `std::terminate` (signal 6, no report).

## Decision

### 1. Unsafe regions ([UNS-1], [GR-10])

`unsafe "reason" { value }` is an expression: its value and type are
`value`'s, the one expression between the braces. The reason is a string
literal that says something (not empty, not only spaces). `unsafe` becomes
a keyword ([LEX-5]); it was a reserved word, so no valid program changes.

Example: the registers of a UART, known from the board's memory map.

```yxy
const UART0: usize := 0x4000_1000

fn main() -> u8
effects {}
{
    regs := unsafe "UART0, 0x4000_1000 to 0x4000_103F, datasheet 4.2" { Mmio.at(UART0, 0x40) }
    send(regs, 65)
    return 0
}

fn send(regs: Mmio, byte: u8)
effects {}
{
    regs.write_u32(0, widen(byte))
}
```

### 2. A local contract, not an effect ([UNS-2], [UNS-3], [UNS-5], [EFF-5])

An unsafe operation is valid only inside a region, and a region must hold
one of its own: the region is exactly the expression whose obligation it
states, never the rest of its function. The region adds no effect and
changes no signature (C3 (b)): `send` and `main` above declare
`effects {}`, and a caller of a function that holds a region writes no
`unsafe`. What a signature no longer says, the tools say: `yxy inspect`
lists every region with its reason and operations (`unsafe_regions`) and
whether each function reaches one (`reaches_unsafe`): it holds one, calls
a function that does, or may destroy a value whose destructor does. Every
call is static in this version, so the call graph, and the fact, are
complete.

[EFF-5] is rewritten: the unsafe boundary is `ffi` and the unsafe
regions, and the memory-safety guarantees hold for a function that **does
not declare `ffi` and does not reach an unsafe region**.

### 3. No safe function dereferences an integer (E-4, [UNS-4])

The operands of an unsafe operation are **fixed in their function**: made
of literals, constants, immutable variables whose value is fixed,
operators, and calls whose arguments are fixed. A parameter is not fixed,
nor a `mut` variable, a variable bound by a pattern or a loop, or one
computed from them; an operand that is not fixed is an error (E0624 of the
compiler). **Nor is a value read through an `Mmio` handle**: device memory
can carry an integer the caller chose. A caller with a handle to the same
block writes an address there, then calls a function whose operand reads
it back (`p := widen(dev.read_u32(0))`, then `Mmio.at(p, 1)`): without
this rule that function would reach an address its caller chose, through
a region whose operands look fixed, which E-4 excludes. The same holds for
what a call may have read through a handle: the result of a call that
receives a handle, and the result of a call of a function that reaches an
unsafe region ([UNS-5]; every call is static, so this is known once the
whole program is checked), are not fixed either. A function that does
need an address from the device receives a handle created where that
address is known (OPEN #51 lists `unsafe fn` for the rest). So
`fn peek(addr: usize) -> u8 { … Mmio.at(addr, 1) … }` is refused
whatever its region says: a function that works on device memory
receives a handle ([MMIO-2]), created where the address is known. The rule
is syntactic and conservative: it reads only what the operand's
expression and the initializers of the variables it names are made of,
and needs no flow analysis, besides the reach of regions over the call
graph that [UNS-5] already computes. A foreign call's result counts as
fixed when its arguments are: what foreign code returns is `ffi`'s, the
other half of the boundary.

### 4. MMIO as a capability ([MMIO-1]–[MMIO-4])

`Mmio` is a prelude type: the authority over a block of `len` bytes at
`base`. `Mmio.at(base, len)` creates one, and is the only unsafe operation
of this version; its region's obligation is that the block is device
memory (or memory valid for these accesses) for the rest of the process,
that no Yxy value lives in it, and that it does not wrap the address
space. Like `Console`, a handle is a parameter or a local, never returned,
stored or passed to C, so that the parameters of a function show every
block it can reach besides its own regions. Its operations,
`read_u8`/`u16`/`u32`/`u64(offset)` and `write_u8`…`u64(offset, value)`,
are safe: each is one volatile access, never removed, duplicated, merged,
split or reordered with another access of a handle of the thread, checked
against the block (*MMIO access out of bounds*) and, wider than a byte,
for alignment (*misaligned MMIO access*). Nothing else is implied: no
barrier, and the order of bytes is the target's. An access performs no
tracked effect in this version: the catalogue of effects grows only
through OPEN #9 (the gate of TASK-20260926-048), and the fact
`reaches_unsafe` and the handle in the parameters already show where
device access happens. The divergence the audit recorded between its row
17 (a capability) and its §20 (an effect of its own) is resolved for the
capability, with the effect left to OPEN #9.

The toolchain's tests reach no hardware: a test hook gives a block of 64
bytes, aligned to 64, and the reference evaluator simulates one at an
address with the same alignment.

### 5. Unwinding: U1 by the personality routine ([ABI-3] (c), [TRAP-1])

Every Yxy function that calls an `extern fn` already had the personality
routine `yxy_rt_personality`, which refused any unwind that reached its
frame. The routine now ends the process with a trap report whatever the
phase — the search for a handler, cleanup, or forced unwinding (glibc's
`pthread_exit` and `pthread_cancel`) — and never returns to the unwinder:
the report is written before any handler above the Yxy frame runs, no
frame is skipped and no destructor runs ([DROP-3]). This is route (ii) of
the study's D5: no declaration or call changes; the study's probe of it
measured +36 bytes of code at -O0 and +196 at -O2 per module on
`aarch64-apple-darwin` (not measured again on this implementation, whose
routine also holds the report's constant), and gave a report in every case
(route (i), a landing pad per call, reported only when a `catch` stood
above the Yxy frame). The routine is one per module, so the
report cannot name the call the unwind came through: its kind is *foreign
unwind reached a Yxy frame*, with no position and no site. In the
compiler's terms: code T0008, the human line
`yxy: trap[T0008]: foreign unwind reached a Yxy frame (site unknown)`,
and a JSON event of a new schema, `yxy.trap/v2`: the fields of
`yxy.trap/v1`, in the same order, with `site`, `file`, `line` and
`column` null. `yxy.trap/v1`, frozen since v0.0.1, does not change: it
stays the event of every failed check, which has a site, and a reader of
it never meets a trap without one (the compiler's implementation decision
0010, amendment of 2026-10-04). `yxy run --json` and `yxy test` report
the event as a trap. A table of call sites read by the routine could name
the call later without changing this rule (OPEN #45 lists it).

Test 5' of TASK-20260926-017, which recorded the end through
`std::terminate`, is replaced in the same change by a test of this
policy: the same C++ exception thrown through each kind of Yxy frame, at
-O0 and -O2, on every runnable target, gives status 101 and that report in
both formats, and a `pthread_exit` under glibc gives it too.

### 6. Ownership across the C boundary ([ABI-4]; OPEN #34)

**Nothing owned or borrowed crosses.** The parameters and results of
`extern fn` and `export fn` stay the values of [ABI-2] (integers, `bool`,
floats), passed by value. So a value with a destructor (an owned string, a
struct) is never given to C nor received from it, and no destructor runs
on the foreign side for a Yxy value; C never frees memory Yxy owns, nor
Yxy memory C owns (it calls foreign code to do so). No view (`&str`,
`&[T]`, `&mut [T]`) crosses, so none outlives its owner on the other side.
`take` means nothing where only copy types cross and is refused as on any
copy type; `inout` cannot cross, since C cannot borrow a caller's place
exclusively. A capability never crosses. An address C gives is an integer,
which Yxy code reaches only through an `Mmio` handle created in a region
whose reason states the obligation. The compiler already refused each of
these (E0311, E0365); this part makes the refusal a rule, with its
reason, and a test that lists every case.

### 7. What the compiler checks

- `tests/fail`: no safe function dereferences an integer (a parameter, a
  value derived from one, a `mut` variable, a call on a parameter, a
  pattern or loop variable, `Mmio.at` outside the region of a function
  that has one elsewhere, an address read back from the device, also
  through a handle whose base foreign code gave, the result of a call
  that receives a handle or of one that reaches a region); the syntax of regions; regions without an
  unsafe operation and operations without a region; the capability's
  places; every owned or borrowed type at the boundary.
- `tests/run`: a handle created in `main` and used by safe functions;
  accesses outside the block and misaligned ones trap at the same site in
  the three engines (the evaluator, -O0, -O2), on every runnable target,
  and in the evaluator on the 32-bit and other 64-bit data models.
- `inspect`: the reach of regions, function by function, against a fixture
  derived by hand (calls, mutual recursion, a destructor, a function that
  only uses a handle, one that only calls foreign code).
- The volatile accesses survive the C compiler's -O2 on every target that
  generates code.

## Alternatives

**Syntax of a region.**
(a) A block of statements, `unsafe "reason" { stmts }`, as a statement:
variables declared inside end with it, so a handle made there could not be
used after it without returning it from a function, which [MMIO-2]
forbids. (b) `unsafe(reason: "…") { … }`, the research's spelling: named
arguments exist nowhere else in the language. (c) An attribute,
`@safety("…") unsafe { … }`: two constructs for one. (d) An optional
reason: a region without one gives a reviewer nothing. Chosen: one
expression with a mandatory reason; statements in a region can be allowed
later (OPEN #51) and only accept more programs.

**Propagation (C3).** (a) `unsafe` as an effect: every user of a driver
would carry it, the contagion `ffi` had. (c) No `unsafe`: MMIO in C, and
`ffi` with two meanings. Chosen: (b), the audit's recommendation.

**Enforcing E-4.** (i) `unsafe fn`, as in Rust: a function whose callers
discharge its obligation; it lets a safe-looking function take an address
only if its callers write `unsafe`, and is the natural next step (OPEN
#51), but adds a second construct now. (ii) A flow analysis of where each
operand's value comes from, through assignments and control flow: more
programs accepted, more to explain and test. (iii) Constant expressions
only: no handle on an address known at run time (from a device tree, or a
test hook). Chosen: operands fixed in their function, a syntactic rule
over immutable bindings, which accepts the addresses of a memory map and
of foreign calls, and refuses every value a caller chooses. For values
read through a handle: (iv) counting them as fixed, as the first form of
this decision did, which the self-check of the task refuted (a caller
writes the address to the device, and the function reads it back);
(v) refusing only an access written in the operand or in the
initializers it names, which a call that reads the device for it
(`Mmio.at(register0(dev), 1)`, or a function that creates its own handle)
still gets around. Chosen: no value read through a handle is fixed,
directly or through a call that receives a handle or reaches a region;
conservative, at the cost of refusing an address computed by a function
that also touches a device.

**MMIO.** (a) Raw pointers and a dereference allowed in a region: what E-4
and the master prompt §8 exclude unless a capability validates the
address. (b) An effect `mmio`: the audit's §20; left to OPEN #9, since the
catalogue does not grow ahead of its own gate. (c) A handle that may be
returned or stored in a struct (a driver struct): more natural for
drivers, but the parameters would no longer show every block a function
reaches; it can be allowed later with a rule for structs (OPEN #51).
Chosen: a capability like `Console`.

**Unwinding.** U1 by (i), a landing pad at each foreign call: every call
becomes `invoke`, +3 996 bytes for 100 calls at -O0, and the report only
when a `catch` stands above the Yxy frame (D5). U2, a mark on the `extern`
that may throw: new syntax, and an unmarked throw still ends through
`std::terminate`. U0, the precondition without a report: what the author's
R-7 replaced. Chosen: U1 by (ii). For the site of the report: (a) the types
of `yxy.trap/v1` with values no check reports (`site`, `line`, `column` 0,
`file` empty): the first form of this decision, retracted by the task's
self-check, because it changed what a `yxy.trap/v1` event can mean while
that schema is frozen since v0.0.1 (rule 4 of the compiler's
`docs/json.md`). Chosen: (b) `site`, `file`, `line` and `column` null,
which changes their types and so is a new schema, `yxy.trap/v2`, written
only for a failure without a site, with `yxy.trap/v1` unchanged.

**Ownership across the boundary.** O2: views lent to C for the duration of
a call, as a pointer and a length, with C's promise not to keep them: what
C APIs that read buffers need, but a promise of the foreign side the
compiler cannot check, and a layout of slices to fix as an ABI. O3: owned
values given to C, with a destructor C calls back: a protocol of ownership
across languages. Chosen: O1, nothing crosses; O2 and O3 stay in OPEN #34,
with the C types of TASK-20260926-070.

## What could change it

- The author's review of E-4's reading ([UNS-4]), of the place of the
  reason, and of `Mmio` as the first unsafe operation.
- The author's answer to R-7.5 at the gate of 0.2: another route, (i) or
  U2, would change part 5 and the test that replaced test 5'.
- Drivers that need a handle in a struct, or an address computed by a
  caller (`unsafe fn`); programs of the freestanding profile (OPEN #41).
- The catalogue of effects (OPEN #9): device access as an effect.
- C APIs that take buffers or strings, measured on the bindings of
  TASK-20260926-072: views lent for a call (O2).
- A call-site table for the report of a foreign unwind.
