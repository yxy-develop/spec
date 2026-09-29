# Decision 0005: Effects in the first subset

- Status: Experimental — plans/decisions/README.md, L-0005
- Date: 2026-09-25
- Spec: `semantics.md` §7, §8
- Errata (2026-09-27): wording only; no rule changed. [EFF-2] no longer
  lists allocation among the effects to come: allocation has facets of an
  effect, a capability and a type, and is decided with ownership
  (`OPEN.md` #6); the classification criterion of `OPEN.md` #9 (a research
  recommendation of the architecture audit of 2026-09-26, experimental) is
  applied to each candidate. `ffi` remains the only tracked effect.
- Amendment (2026-09-28, experimental): decision 0015 adds a second tracked
  effect, `console`, performed by the operations of the first capability,
  `Console` (`semantics.md` [EFF-2], [CON-3]). Item 2 below reads "`ffi`
  and `console`"; the other items do not change.

## Decision

1. Every function declares `effects { … }`; the clause is mandatory, and
   `effects {}` means no tracked effect (it does not exclude traps, cost or
   non-termination).
2. The only tracked effect is `ffi`: a call into code outside Yxy. Every
   `extern fn` must declare it. Foreign effects are trusted declarations.
3. At each call the callee's declared effects must be contained in the
   caller's. The check uses declarations, so it covers recursion.
4. Traps, typed failures and local mutation are not effects.

## Alternatives

- A closed catalogue such as `{console, test}` (independent draft). Deferred:
  without a standard library there is no console or test resource to name, and
  `ffi` is the one boundary that actually exists.
- Inferring effects instead of declaring them. Rejected: signatures must state
  their effects so that a reader does not need the callee's body.

## What could change it

The first standard library: effects and capabilities for console, files,
network, clock, randomness and allocation, and how `main` receives them.
