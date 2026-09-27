# Decision 0013: Guarantees at the C boundary

- Status: Accepted — **experimental** (proposed in the conformance phase that
  follows the architecture audit of 2026-09-26, tasks TASK-20260926-014, 015,
  016, 017 and 024; open to the author's review)
- Date: 2026-09-26; item 5 amended 2026-09-27
- Spec: `semantics.md` [TRAP-1], [EFF-5], [ABI-2], [ABI-3]; `OPEN.md` #44,
  #45 (numbers 41 to 43 were held for entries planned elsewhere, written on
  2026-09-27)
- Amendment (2026-09-27, conformance fix round, experimental): item 5 now
  names the symbols the assumption of [EFF-5] covers. Once the compiler made
  copies and fills without any external call (the implementation's R117),
  "the linked objects do not define other C library functions that the
  generated code calls" was wider than needed and suggested that a linked
  `memcpy` could still change what code without `ffi` does; the assumption is
  now the hosted runtime's symbols and the `__…` helpers of the C
  implementation, and [EFF-5] states that copies and fills call nothing a
  program or a linked object can define. No other item changes.

## Context

The architecture audit of 2026-09-26 measured the C boundary against the
specification and found five places where the text promised more than the
implementation kept, or was wrong for some target:

- **Library calls of the code generator.** A struct copy is lowered to
  `memcpy`, and `[value; N]` to `memset`, `bzero` or `memset_pattern16`
  depending on the target and the optimization level. [ABI-3] reserved only
  `main`, `write`, `_exit`, `getenv`, `yxy_rt_…` and `__…`, so a program
  could define `export fn memcpy`: the copies of functions declared
  `effects {}` then ran user code and, in the audit's probes, read
  uninitialized memory (exit 255 at -O0 against 63 at -O2 and in the
  reference evaluator). An object given with `--link` can do the same without
  any `export`.
- **Narrow integers.** [ABI-2] said that values narrower than 32 bits "are
  extended by the caller". That holds for Apple's ARM64 ABI, not for the
  generic AAPCS64 used by Linux on ARM64, where the bits beyond a parameter's
  width are unspecified. With a conforming C caller that leaves them dirty, an
  `export fn` indexing a table with a `u8` parameter read outside the table at
  -O2.
- **Unwinding.** [ABI-3] said that unwinding across the boundary "does not
  exist", while Yxy functions and `extern` declarations are compiled as never
  unwinding (`nounwind`), and a C++ exception did cross a Yxy frame in a
  probe. Declaring code `nounwind` when it may unwind is undefined behaviour
  of the backend.
- **One report per trap.** [TRAP-1] promises one report, and 8 threads
  of a C host trapping in Yxy code at the same time wrote 6 to 8 reports.
- **The assumption behind [EFF-5].** "Memory-safe without `ffi`" silently
  assumed that nothing linked into the program replaces what the generated
  code calls.

## Decision

1. **One reservation, independent of the target ([ABI-3] (a), (b)).** The
   hosted runtime's symbols, `yxy_rt_…` and `__…` stay reserved for `extern`
   and `export`. The C library functions the code generator may call on its
   own — `memcpy`, `memmove`, `memset`, `memcmp`, `bcmp`, `bzero`,
   `memset_pattern16` — and names starting with `_` are reserved for
   `export`. C11 7.1.3 reserves the names starting with `_` for the
   implementation as identifiers with file scope, which an `export` symbol is
   (`_start`, the entry point of ELF programs; `_mh_execute_header` of Mach-O
   executables; `_GLOBAL_OFFSET_TABLE_`, which 32-bit x86 objects refer to):
   an `export fn _start` passed `yxy check` and then failed at link time
   (duplicate symbol). The same clause also reserves every external
   identifier of the standard library (`exit`, `malloc`); only the names
   above are reserved here, and the rest is open (`OPEN.md` #44). Reserving
   every name that starts with `_`, and not only the names the code
   generator uses (the list of the audit), is a widening of this experimental
   decision that awaits confirmation by the Orchestrator or the author; it
   refuses names such as `_helper`, which no linker would reject. The list is
   the same on every target,
   including those that never call a given name: a program's validity may not
   depend on the target except through the range of `usize`/`isize`
   ([TGT-2]). The reservation is a second defence: the compiler should
   generate copies and fills that do not depend on a replaceable symbol where
   its backend allows it.
2. **`extern` of those names stays accepted.** `extern fn memcpy(…) effects {
   ffi }` only declares the C library's function; a call to it is an ordinary
   `ffi` call, visible in the caller's effects. It does not change what the
   generated copies call (the audit's probe with such a declaration returned
   the right value in every engine). Refusing it would forbid a legitimate
   use of the C library and would protect nothing. The dangerous case is a
   *definition* of the symbol, which only `export` (or a linked object)
   provides.
3. **Narrow integers follow each target's ABI and are normalized on
   receipt ([ABI-2]).** When Yxy passes a narrow value to C it extends it as
   the target's ABI requires; when it receives one (a parameter of an
   `export fn`, the result of an `extern fn`) the Yxy side normalizes it to
   its type, so that Yxy code never sees bits outside the value's width, on
   every target.
4. **No unwinding across Yxy frames is a precondition of `ffi` ([ABI-3]
   (c)).** Foreign code must not unwind into or across Yxy frames (C++
   exceptions, forced unwinding, `longjmp`); C++ code called from Yxy catches
   its exceptions before returning. Compiling Yxy functions and `extern`
   declarations as never unwinding is then a consequence of the contract, not
   an unverified claim. When the precondition is broken through the
   unwinder, the frame of the Yxy function that called the foreign code
   refuses the unwind (the personality routine of such a function refuses
   every unwind that reaches it), so the process ends (C++:
   `std::terminate`) instead of skipping Yxy frames. The specification
   promises this only where that frame is on the stack while the foreign
   code runs. A `longjmp`, or the end of a thread that does not unwind, is
   not detected, and breaking the precondition that way is outside the
   guarantees. Whether the end becomes a trap report, or unwinding a
   declared contract, is decided with destructors (`OPEN.md` #45).
5. **The assumptions of [EFF-5] are stated.** Its guarantees hold when
   foreign callers follow the C ABI, foreign code does not unwind across Yxy
   frames, and the linked objects do not define the C symbols that the
   generated code calls: the hosted runtime's `write`, `_exit` and `getenv`,
   and the helpers of the C implementation whose names start with `__` (the
   stack probe, the arithmetic helpers of 32-bit targets). The code generated
   to copy or fill memory calls no function that the program or a linked
   object can define, so a linked `memcpy` or `memset` does not reach it
   (amended 2026-09-27; before, the assumption also covered "other C library
   functions that the generated code calls"). The compiler checks what it can
   (`export` of reserved names); it does not check linked objects.
6. **One trap report per linked image ([TRAP-1]).** When several threads
   trap at the same time, the first trap of a linked image (an executable or
   a shared library, with every Yxy object linked into it) writes its report
   and ends the process; the other traps of that image write nothing. Yxy
   code in two shared libraries of one process forms two images, each of
   which may write one report.

## Alternatives

- **Reservation per target** (only the names a target's code actually
  calls). Rejected: validity would depend on the target, against [TGT-2], and
  the set changes with the optimization level and the LLVM version.
- **Reserving every external identifier of the C standard library.** The
  principled rule, but much wider (`exit`, `malloc`, `time`…) and not needed
  to close the defect found; left open (`OPEN.md` #44).
- **Renaming exported symbols, or requiring `ffi` on `export fn`.** The first
  breaks [ABI-1] (an `export fn` is callable from C under its own name); the
  second changes only a label and protects nothing.
- **Refusing `extern` of the reserved library calls too.** See item 2: it
  forbids a harmless and legitimate declaration.
- **Not claiming `nounwind`** (letting foreign exceptions pass through Yxy
  frames). Without destructors nothing would run during unwinding, but it
  would make unwinding a behaviour to specify and test before there is a use
  for it; decided with destructors instead.
- **Leaving an exception that reaches a Yxy frame undetected** (only the
  precondition, no personality routine). Rejected: the backend's `nounwind`
  would then be broken silently; refusing the unwind turns the violation into
  the end of the process, as Rust's `extern "C"` functions do since 1.81.
- **"The caller extends" kept as the rule, documented as a precondition of
  foreign callers.** Rejected: a conforming AAPCS64 caller would break it,
  and the consequence (a read out of bounds in code without `ffi`) is a
  memory-safety failure on the Yxy side.
- **One report per thread, documented** instead of one per linked image.
  Rejected: foreign threads can already enter Yxy code, so [TRAP-1] is
  testable today, and a single report keeps the contract of the structured
  report (`yxy.trap/v1`: one line).
- **One report per process, across linked images.** It needs a state shared
  by every image of the process, that is a symbol each image exports and
  foreign code can see or replace. Not adopted in this version, where Yxy
  code in several shared libraries of one process is not a supported case;
  the scope is stated instead.

## What could change it

A backend that guarantees copies and fills without any external call on every
target (the reservation of item 1 could then shrink); the design of
destructors and ownership (item 4); a C library or target whose ABI needs
another normalization rule (item 3); concurrency in the language itself, or
programs with Yxy code in several shared libraries (item 6).
