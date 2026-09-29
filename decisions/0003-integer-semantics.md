# Decision 0003: Integer semantics and traps

- Status: Experimental — plans/decisions/README.md, L-0003
- Date: 2026-09-25
- Spec: `semantics.md` §6

## Context

The bootstrap instructions require the numeric policy to be fixed before code
generation: literal typing, conversions, overflow, division by zero, signed
minimum divided by −1, invalid shifts and indices; defined behaviour with
detected failure, identical in debug and release; checked operations returning
an error or absence; wrapping and saturation explicit.

## Decision

1. An arithmetic operation traps exactly when its mathematical result is not
   representable or is undefined (shifts and bitwise operations have their own
   rules: a shift traps only on an invalid amount and `<<` discards bits): `+ - *` and unary `-` on overflow, `/` and
   `%` by zero, signed `MIN / -1`, shifts by an amount outside `0..bits`, and
   out-of-bounds indexing. `MIN % -1` is `0` (representable), not a trap.
2. The rules are identical at every optimization level; there is no unchecked
   mode. The generated code never carries `nsw`, `nuw`, `inbounds`, `exact` or
   assumptions the analysis did not prove.
3. `checked_add/sub/mul/div/rem` return `Option`; `wrapping_add/sub/mul` wrap
   explicitly. Saturating operations are not provided yet.
4. Integer literals have no default type; the type comes from context. There
   are no implicit conversions. `widen(x)` converts only when every value fits
   (checked at compile time); `checked_convert(x)` returns `Option` (checked at
   run time). There is no truncating cast.
5. A trap writes one report to standard error and exits with status 101,
   without cleanup. Since 2026-09-25 the report carries a stable code, the
   site of the check and the source position, as a line of text or, when the
   program's environment asks for it, one JSON line ([TRAP-1], [TRAP-3];
   compiler implementation decision 0005). Before that it was
   `yxy: trap: <kind> at <file>:<line>:<column>`.

## Alternatives

- Wrapping in release builds (C, Rust release). Rejected: behaviour would
  depend on the build mode.
- Trap by `llvm.trap` (SIGTRAP). Verified to work, but every trap creates a
  macOS crash report and the process is killed by a signal. Exiting with a
  message and a fixed status keeps traps observable, deterministic and cheap to
  test. A debugger-friendly mode (stop at the faulting instruction) is an open
  item.
- A default integer type for literals (Go `int`, Rust `i32`). Rejected: a
  hidden choice that changes range and overflow behaviour.

## What could change it

Measured cost of the checks in real workloads (then: proven elimination, not a
silent mode); a need for traps visible to debuggers; experience with the
verbosity of `widen`/`checked_convert`.
