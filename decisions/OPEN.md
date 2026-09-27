# Open language decisions

Each entry: the provisional choice in force, its risk, and the evidence that
would change it. Closing an entry means writing a numbered decision. Numbers
41 to 43 were held for entries planned elsewhere (the master plan of
2026-09-26, proposals MP-a1, MP-b2 and MP-c) and were written on 2026-09-27;
the entries added meanwhile start at 44. Entries 41 to 43 are open questions
only: no rule of the specification changes with them.

| # | Question | Provisional choice | Risk | What would change it |
|---|---|---|---|---|
| 1 | Region statement restrictions ([CELL-2]) | `require` in `@ctrl` and, after the computation, in `@eval` (author decision A1, 2026-09-26, experimental); `return` only in `@out`; calls with effects allowed in `@ctrl` | invents rules the author did not state; a call with effects in a declaration of `@ctrl` runs before a false guard ([CELL-6]) | author review; awkward real programs; the alternative that restricts the calls of `@ctrl` to functions without effects, so that a false guard prevents every effect (decision 0004) |
| 2 | Formatter layout of cells | always multi-line | loses the compact presentation the author also shows | author preference; readability study |
| 3 | `when` cells | rejected | the pattern is incomplete without it | a design for skipped cells and `Option` results |
| 4 | Nested cells and block values | not allowed | limits composition | a construct for leaving a nested cell that is not `return` |
| 5 | Enum payloads, generics, traits | not supported | most real programs need them | milestone 0.1/0.2 designs |
| 6 | Ownership, moves, `&mut`, cleanup order | not supported (all types copy) | memory-safety claims limited to the subset | milestone 0.2 |
| 7 | Text: bytes, borrowed UTF-8, owned strings | not supported | no text processing | a design separating bytes, code points and graphemes |
| 8 | Floating point | not supported | no numeric code | a policy for NaN, signed zero, rounding and no fast-math by default |
| 9 | Effect catalogue and capabilities for `main` | only `ffi`; `main` gets any effect; *(experimental)* each candidate concern is classified as a type, a contract, an effect or a capability by the four questions and the tie-break rules of the note below | effects say little yet | the first standard library |
| 10 | Unicode identifiers | ASCII only | excludes non-English identifiers | UAX #31 policy with confusable detection |
| 11 | Trap mechanism | message + exit status 101 | debuggers do not stop at the check | a debugger-friendly build option |
| 12 | Concurrency | not supported | — | structured concurrency design (milestone 0.5) |
| 13 | Saturating arithmetic | not provided | — | a use case |
| 14 | Enums with more than 256 variants | rejected | large generated enums | a wider tag, chosen by variant count |
| 15 | Unit of a package: directory or file | a directory of `.yxy` files inside a module; a standalone file is a one-file package (`modules.md` [PKG-1], [PKG-7]) | finding a name may need the other files of the directory | a reading or agent study comparing both; author preference |
| 16 | Package clause | `package <name>` replaced `module <name>` (implemented 2026-09-26; `module` is rejected with a mechanical fix) | every file changes once | author preference for keeping `module` |
| 17 | Manifest and lock format | the layout of compiler implementation decision 0006: a strict TOML subset; `dir` for local replacements; optional `license` in `[module]`; lock format 1 with `main`, `[require]`, `[replace]`, `[[module]]`, `[[unselected]]` (unselected versions record their requirements); content hash `yxy-content-v1`; limits 20 000 entries, 256 MiB, 1024-byte paths; an unpublished module is required at `0.0.0` and replaced by its directory | early tools bind to an interim format | the first implementation of `lock` and `fetch`; author review |
| 18 | Major versions | a `/vN` path element for N ≥ 2; majors coexist as distinct modules ([VER-4]) | path changes at every major; repository layout conventions | experience with the first published modules; author preference |
| 19 | Version selection | minimal version selection, experimental ([VER-5]); majors 0 and 1 are one module ([VER-4]), so selection moves a build from `0.x` to a later `0.y`, or from 0 to 1, like any higher version, without a warning; the requirements of every module reached count, also those of modules that no package of the build imports | relies on compatibility within a major that nothing checks; no upper bounds; `0.x` modules, which promise nothing ([VER-3]) and are most of a young ecosystem; modules reached only through requirements ("phantom" modules) can raise versions without being used | breakages in real dependency graphs; a public-interface compatibility checker; for `0.x`: a note in the diff of the lock when selection crosses a `0.x` minor or goes from 0 to 1 above someone's minimum, or treating a divergence of `0.x` minors as a conflict until the main module fixes the version; for phantom modules: pruning the graph to the modules the build imports (Go prunes it since go 1.17); author decision |
| 20 | Trust and signatures | lock hashes only; a first acquisition is trusted when the user accepts the diff ([NET-5]) | a first download is not checked against an independent reference; no author identity | a checksum-log or signature design, together with the service |
| 21 | Private origins | patterns are path elements or `*` (one whole element), matching every path whose first elements they match; the first element is an origin or `*`; applied before the first request, with no fallback ([NET-3], [NET-4]) | pattern syntax, credential helpers, leaks through name lookups or logs | a design and tests with a private origin |
| 22 | Index/proxy protocol; repository root of an origin | none: direct access to origins and local directories; root known for some hosts, otherwise an explicit source ([PATH-7], [MAN-5], [NET-2]) | dependence on origin availability; custom domains need an explicit source | the versioned HTTP protocol of the service |
| 23 | Repository, name and owner of the package service | none created — **awaiting the author** | service work without a home | the author's decision, proposed through `plans` |
| 25 | Untagged revisions (pseudo-versions) | cannot be required; a replacement is used ([VER-2]) | friction to use an unreleased fix | demand from real use |
| 26 | Target-specific files and requirements | none: every file and requirement on every target ([PKG-6], [BLD-4]) | platform code needs another structure | standard library and FFI needs per target |
| 27 | Native code in dependencies | not supported; foreign code linked only by the main build ([INIT-4]) | libraries that wrap C cannot be distributed | a declared native-build contract with no arbitrary scripts |
| 28 | Visibility within a module; struct fields | every `pub` item is visible to every importer, no module-internal packages; struct fields private unless `pub` ([VIS-2]; implemented, with [VIS-3] applied to `pub` fields too) | every `pub` item of every package is public interface | the struct design (TASK-20260925-002); multi-package libraries |
| 29 | Selective imports, re-exports, workspaces | none: qualified names only; local development by path replacements ([IMP-6], [MOD-4]) | verbosity; repeated replacements across local modules | experience with real multi-module projects |
| 30 | Standard library versioning | supplied by the toolchain at its version; manifests state a minimum language version and nothing is downloaded ([STD-1], [MAN-7]) | library changes tied to compiler releases | the first standard library |
| 31 | Letters in import paths | lowercase ASCII only ([PATH-2]) | origins with uppercase names need an explicit source | origins where lowercase paths cannot be obtained |
| 32 | Struct equality and ordering | none; compare fields ([STRUCT-7]) | field-by-field comparisons can miss a field | traits or derived equality; floating point |
| 33 | Struct patterns and destructuring | not supported; `match` on a struct is an error, and `Name { … }` in a pattern is rejected (the compiler's E0900) | nested field reads instead of patterns | a pattern design with exhaustiveness over fields, together with #46 |
| 34 | C ABI for structs | never cross the boundary; layout internal ([STRUCT-8]) | no interop with C APIs that take or return structs | an explicit C-layout opt-in with per-target by-value/by-pointer rules |
| 35 | Struct layout optimizations | declaration order, natural padding; `Option`/`Result` as `{ tag, … }`, no niches, `Result` stores both payloads | wasted space | measurements; field reordering or niche tags (allowed: the layout is internal) |
| 36 | Size limits of value types ([STRUCT-9]) | at most 16 384 scalar components and 256 levels of nesting per struct, `Option` or `Result` type | rejects legitimate large aggregates | measurements on real programs; code generation whose cost does not grow with the number of components |
| 37 | Assigning fields of array elements and temporaries | rejected; replace the element (`a[i] = S { … }`) | awkward updates of arrays of structs | place paths with index and field; an order rule for `a[i].f = v` |
| 38 | Struct literals in `if`/`while`/`match` heads ([GR-6]) | must be parenthesized (Go/Rust rule) | surprises new readers | reader/agent studies; a literal syntax without the ambiguity |
| 39 | Empty structs | rejected (like enums without variants) | no marker or unit-like types | a use case (capabilities, typestate) |
| 40 | Field shorthand and update syntax | not supported | verbose literals | usage evidence |
| 41 | Freestanding profile: trap report and stack exhaustion without an operating system | none: code generation is refused for targets without an operating system (the compiler's `TARGETS.md`); [TRAP-1] and [TRAP-2] describe hosted behaviour only | the `core`/freestanding profile of [STD-2] (`modules.md`) has no trap or stack contract; 32-bit ARM and bare metal stay analysis-only | the author's trap policy (decision A4 of the control repository; #11); a profile design with a trap hook and a stack bound (static bound over a recursion-free call graph, MPU guard or stack-limit registers) |
| 42 | Size in bytes of a value | no limit in bytes: a value larger than the stack or the address space is accepted and ends the process when its frame is entered ([TRAP-2]) | a program that can never run on a 32-bit target is accepted silently (`[u32; 2000000000]` on i686) | an author decision between a target-dependent limit (it would amend [TGT-2]) and a portable one; measurements |
| 43 | Optimization of checked operations: literal or as-if reading of [TRAP-3] | literal text; the compiler adds no check transformation of its own (LLVM already computes an operation together with its overflow flag before branching to the trap) | the literal text forbids transformations with the same observable behaviour (block-wise checks with exact redo, vectorized loops); an as-if reading needs an exact list of what is observable: output and foreign calls before the trap, in order; the report (code, site, position); exit status 101; memory reachable by foreign code | a measured optimization that keeps the first failing check, its report and every earlier effect; author decision. Any proposal of code generation that optimizes checks cites this entry |
| 44 | Names of the C library as `export` symbols; `export` as a boundary | reserved: the hosted runtime's symbols, the C library functions the code generator may call, `__…` and every name starting with `_` ([ABI-3]; decision 0013; reserving every `_…` name, beyond the names the code generator uses, is part of that experimental decision and awaits confirmation) | an `export fn` named like another C library function (`exit`, `malloc`) replaces it for the foreign code and the C runtime of the program, without `ffi`; C11 7.1.3 reserves those names too, so the reservation cites that clause only in part | reserving every external identifier the C standard reserves; an explicit marker or effect for `export`; a checker of symbols at link time; a use of `export` names starting with `_` (then only `_` + uppercase and the names of the implementation) |
| 45 | Unwinding across the C boundary | a precondition of `ffi`: foreign code never unwinds across Yxy frames, which are compiled as never unwinding; an unwind that reaches the frame of a Yxy function that called foreign code is refused there and the process ends (C++: `std::terminate`), where that frame is on the stack; a `longjmp`, or the end of a thread that does not unwind, is not detected and is outside the guarantees ([ABI-3] (c); decision 0013) | the end of the process is not a trap report ([TRAP-1]); with destructors, a foreign exception or a `longjmp` would skip cleanups | the design of destructors and ownership (#6): a trap report at the boundary, or a declared unwinding contract |
| 46 | Or-patterns, match guards and range patterns (with #33) | not supported: `p \| q`, `pattern if condition` and `a..b` or `a..=b` in a pattern are rejected, one error per arm (the compiler's E0900), and the `match` is read on | a value handled alike in several arms is written once per arm; a condition moves into the arm (a `match` on it), where exhaustiveness does not see it; an integer `match` needs `_` ([MATCH-1]) | a pattern design together with #33 and enum payloads (#5): or-patterns that bind the same names with the same types in every alternative; guards that do not count for exhaustiveness; integer ranges with exhaustiveness over intervals, whose check splits the ranges into disjoint intervals and costs more than today's single column of constructors ([GR-5] bounds the input, not that cost); a measured bound for generated matches |
| 47 | Sub-slices `s[i..j]` | not supported: a slice views a whole array ([REF-1]); `s[i..j]`, `s[..j]`, `s[i..]` and `s[i..=j]` are rejected (the compiler's E0900), and there is no range expression | a function that works on part of an array takes the slice and the bounds as separate parameters, and each index is checked against the whole length, so an index outside the part is not caught | a design that states the bounds check (`i <= j` and `j <= len`, one trap kind or two, its site as in [TRAP-3]), the cost (two comparisons per sub-slice; the slice keeps its two-word form), the open and inclusive forms, whether `..` becomes a range expression (with `for`), and slices of `mut` arrays (#6); places with projections in the compiler (#37) |
| 48 | Evolution of the language after it opens | *(experimental; awaiting the author, with the decision to open the repositories)* until 1.0, every change that makes a valid program invalid or changes its meaning raises the language version (the version the toolchain implements, and the minimum a manifest requires, `yxy = "0.N"`, `modules.md` [MAN-7]) and comes with a mechanical fix: a diagnostic whose fix, applied to the old program, gives a program of the new version with the same meaning, as `module` became `package` ([MIG-1]). A change that only accepts more programs needs neither | [MAN-7] checks a minimum and never selects behaviour, so a toolchain of the new version reads old code with the new rules: it reports the changed constructs, with their fixes, instead of compiling them with their old meaning; a standalone file ([PKG-7]) has no manifest, so no version; a change that keeps the version number makes that number ambiguous. Changes that are cheap before the opening and need this rule after it: reserving `let` and `var`, identifiers today ([LEX-7]); making `match` a statement ([GR-3]); making `;` a terminator ([LEX-16]) | the author's decision, with the opening; editions selected per module, when a break after 1.0, or a change that has no mechanical fix, requires them |

## Note on #9: type, contract, effect or capability *(experimental)*

A research recommendation of the architecture audit of 2026-09-26 (§12b.1.2,
in the control repository), recorded here as the working criterion for the
first standard library. It is not a rule of the specification and changes no
rule; the author may replace it.

For each concern, ask these questions in this order; the first "yes" gives
its class:

1. Is it a value that the caller receives and can inspect or handle? Then it
   is a **type** (failure, absence, ownership, mutability, exit status).
2. Is it a predicate on values that must hold at a point, whose violation
   becomes a typed failure or a trap? Then it is a **contract** (`require`, a
   postcondition, a trap check, the obligation of `unsafe`, the order of the
   regions, a transaction). A contract grants no authority and describes no
   action.
3. Can it happen during the call, and must a caller be able to exclude it
   transitively by reading only the signature? Then it is an **effect**. An
   effect names the kind of action, never the resource it uses. A name enters
   the catalogue of the core only if the language itself, or the hosted
   runtime, mediates the action.
4. Can two calls with the same effect reach different resources, and does the
   difference matter (tests, isolation, security)? Then it is a
   **capability**: a value that carries authority over a concrete resource,
   obtained from `main` or from another capability, never from a global
   singleton.

Tie-break rules:

- A concern may have facets in several classes. The primary class is
  recorded and the others are cited (writing to a file: the capability says
  which file, the effect `fs` says that the function may touch the file
  system, the type carries the I/O error in a `Result`).
- The noise test: adding an effect to a `pub fn` is an incompatible change
  ([VIS-5], [VER-3]), so an effect that almost every function would declare
  says nothing and only causes breakage. Before creating an effect, count the
  signatures of a corpus that would carry it.
- A "capability of the target" (does the target have a file system, an
  operating system?) is not a capability: it belongs to a profile or a target
  ([TGT-1]: the target does not change effects; [STD-2]: a freestanding
  program imports only `core`).
- Cost, termination and traps are not effects ([EFF-1]); a predictable cost
  belongs to a cost report, not to the signature.
- Ownership stays within types until ownership is designed (#6).
