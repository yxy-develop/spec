# Decision 0010: Multi-architecture and target-dependent integers

- Status: Author (item 1), Experimental (items 2–4) — plans/decisions/README.md, L-0010.
  Item 1 was decided by the author on 2026-09-25, conveyed through the
  portability review in the control repository.
- Spec: `semantics.md` [TY-2], [CONV-1], [TGT-1]–[TGT-3]

## Decision

1. **Yxy is multi-architecture by design**: 32- and 64-bit systems, x86,
   x86-64, 32-bit ARM, ARM64, RISC-V, and other families added over time. The
   first validated environment (macOS on ARM64) is not a limit of the
   language. A target counts as supported only with implementation and
   evidence for it.
2. `usize` and `isize` have the pointer width of the target's data model — 32
   or 64 bits in this version. All other integers have a fixed width on every
   target.
3. The **validity** of a program does not depend on the target, except for the
   range of `usize`/`isize` literals: `widen` is accepted only when the
   conversion is lossless on every supported data model (`u32 → usize` yes,
   `u64 → usize` no, `usize → u64` yes). Conversions that may lose values on
   some target use `checked_convert`.
4. The target is always explicit inside the compiler: the checker, the
   evaluator, the code generator, the driver and the reported facts receive
   the same target. The host never decides the meaning of a program silently.

## Alternatives

- Target-dependent `widen` (accept `u64 → usize` on 64-bit targets): the same
  source would compile for one target and fail for another for reasons the
  author of the code cannot see locally. Rejected.
- A single 64-bit model everywhere: excludes 32-bit systems. Rejected by item 1.

## What could change it

A 16-bit target (would lower the minimum pointer width); segmented or
capability hardware where pointer and index widths differ.
