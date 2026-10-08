# Decision 0031: Versioning a block of checked work behind a guard

- Status: Author — decided by Paulo R. Lima on 2026-10-08, answering
  `decisions/OPEN.md` #43 for the optimization this decision describes.
- Date: 2026-10-08
- Origin: `decisions/OPEN.md` #43 ("Optimization of checked operations:
  literal or as-if reading of [TRAP-3]"); decision R-16 of the control
  repository (performance before 0.1.0); the compiler's work on checked loops
  (`compiler/src/hoist.rs`).
- Spec: `semantics.md` [TRAP-3] (amended), [TRAP-1], [ORD-1]–[ORD-6].

## Context

[TRAP-3] forbids optimization from removing "a check whose failure is
possible", moving it after effects that follow it in evaluation order, or
merging the reports of two checks. OPEN #43 asked whether that text is read
literally, which would forbid every transformation of checks, even one with
the same observable behaviour, or *as if*, which needs an exact list of what
is observable.

A loop that checks every addition and every index cannot be vectorized as it
stands, because each element has a branch to its own trap. In Yxy every build
keeps its checks, so that cost is paid in every program. The compiler's
measurements showed a checked reduction at about five times the cost of the
same loop in C or Rust with the checks turned off (benchmark B0).

## Decision

1. **Guarded versioning is within the literal [TRAP-3].** The compiler may
   split a run of checked work, such as a loop, into blocks. Before each
   block it evaluates a *guard*, and the guard:
   - has no effect;
   - reads only memory the block itself would read, and only at indices the
     guard checks first;
   - proves that no check of the block can fail.

   When the guard holds, the block runs a copy with those checks absent.
   When it does not hold, the block runs the original code with every check
   in place, so its first failing check traps exactly as before.

   In this scheme:
   - no check whose failure is possible is removed, because a check is absent
     only on a path where its failure has been proven impossible;
   - no check moves after an effect;
   - no two reports are merged;
   - no operation runs twice;
   - the output, the foreign calls, the first failing check, its report
     (code, site, position) and the exit status are those of the program
     without the transformation.
2. **Speculation remains outside the literal reading.** A transformation
   that runs work without its checks and redoes it with them when one would
   have failed ("block-wise checks with exact redo") still needs the as-if
   reading. OPEN #43 keeps that question open.
3. **The guard is the compiler's.** It adds no undefined behaviour of the
   backend: no `nsw`, `nuw`, `inbounds` or assumption of the kind [TRAP-3]
   forbids. Its arithmetic is itself checked or done with overflow flags.
   Any guard that cannot be built conservatively leaves the loop unchanged.
4. **Disclosure.** The compiler documents which loops it versions and how a
   test can turn versioning off. Turning it off is never an unchecked mode.
   The facts of `yxy inspect` (checks, sites) describe the program as
   written, and versioning does not change them.

## Consequences

- [TRAP-3] gains one sentence that cites this decision.
- OPEN #43 is answered for guarded versioning; its question about
  speculation with redo stays open.
- The compiler may publish the measurements of guarded versioning under the
  publication conditions of its benchmarks.
