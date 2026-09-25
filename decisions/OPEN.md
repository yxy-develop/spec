# Open language decisions

Each entry: the provisional choice in force, its risk, and the evidence that
would change it. Closing an entry means writing a numbered decision.

| # | Question | Provisional choice | Risk | What would change it |
|---|---|---|---|---|
| 1 | Region statement restrictions ([CELL-2]) | `require` only in `@ctrl`, `return` only in `@out` | invents rules the author did not state | author review; awkward real programs |
| 2 | Formatter layout of cells | always multi-line | loses the compact presentation the author also shows | author preference; readability study |
| 3 | `when` cells | rejected | the pattern is incomplete without it | a design for skipped cells and `Option` results |
| 4 | Nested cells and block values | not allowed | limits composition | a construct for leaving a nested cell that is not `return` |
| 5 | Structs, enum payloads, generics, traits | not supported | most real programs need them | milestone 0.1/0.2 designs |
| 6 | Ownership, moves, `&mut`, cleanup order | not supported (all types copy) | memory-safety claims limited to the subset | milestone 0.2 |
| 7 | Text: bytes, borrowed UTF-8, owned strings | not supported | no text processing | a design separating bytes, code points and graphemes |
| 8 | Floating point | not supported | no numeric code | a policy for NaN, signed zero, rounding and no fast-math by default |
| 9 | Effect catalogue and capabilities for `main` | only `ffi`; `main` gets any effect | effects say little yet | the first standard library |
| 10 | Unicode identifiers | ASCII only | excludes non-English identifiers | UAX #31 policy with confusable detection |
| 11 | Trap mechanism | message + exit status 101 | debuggers do not stop at the check | a debugger-friendly build option |
| 12 | Concurrency | not supported | — | structured concurrency design (milestone 0.5) |
| 13 | Saturating arithmetic | not provided | — | a use case |
| 14 | Enums with more than 256 variants | rejected | large generated enums | a wider tag, chosen by variant count |
| 15 | Unit of a package: directory or file | a directory of `.yxy` files inside a module; a standalone file is a one-file package (`modules.md` [PKG-1], [PKG-7]) | finding a name may need the other files of the directory | a reading or agent study comparing both; author preference |
| 16 | Package clause | `package <name>` replaces `module <name>` when packages are implemented; `package` reserved now ([MIG-1], [MIG-2]) | every file changes once | author preference for keeping `module` |
| 17 | Manifest and lock format | the contract of `modules.md` §6; TOML layout, field names, lock format version, content-hash encoding, extraction limits and the spelling of an unpublished local module are not fixed | early tools bind to an interim format | the first implementation of `lock` and `fetch`; author review |
| 18 | Major versions | a `/vN` path element for N ≥ 2; majors coexist as distinct modules ([VER-4]) | path changes at every major; repository layout conventions | experience with the first published modules; author preference |
| 19 | Version selection | minimal version selection, experimental ([VER-5]) | relies on compatibility within a major that nothing checks; no upper bounds; `0.x` modules | breakages in real dependency graphs; a public-interface compatibility checker; author decision |
| 20 | Trust and signatures | lock hashes only; a first acquisition is trusted when the user accepts the diff ([NET-5]) | a first download is not checked against an independent reference; no author identity | a checksum-log or signature design, together with the service |
| 21 | Private origins | path patterns in the main manifest and in the user's configuration, applied before the first request, with no fallback ([NET-3], [NET-4]) | pattern syntax, credential helpers, leaks through name lookups or logs | a design and tests with a private origin |
| 22 | Index/proxy protocol; repository root of an origin | none: direct access to origins and local directories; root known for some hosts, otherwise an explicit source ([PATH-7], [MAN-5], [NET-2]) | dependence on origin availability; custom domains need an explicit source | the versioned HTTP protocol of the service |
| 23 | Repository, name and owner of the package service | none created — **awaiting the author** | service work without a home | the author's decision, proposed through `plans` |
| 24 | Network during builds | builds never use the network; `yxy fetch` obtains pinned content ([DEP-7]) | one more step after cloning | author preference for builds that download pinned, verified content and report it |
| 25 | Untagged revisions (pseudo-versions) | cannot be required; a replacement is used ([VER-2]) | friction to use an unreleased fix | demand from real use |
| 26 | Target-specific files and requirements | none: every file and requirement on every target ([PKG-6], [BLD-4]) | platform code needs another structure | standard library and FFI needs per target |
| 27 | Native code in dependencies | not supported; foreign code linked only by the main build ([INIT-4]) | libraries that wrap C cannot be distributed | a declared native-build contract with no arbitrary scripts |
| 28 | Visibility within a module; struct fields | every `pub` item is visible to every importer, no module-internal packages; struct fields private unless `pub` ([VIS-2]) | every `pub` item of every package is public interface | the struct design (TASK-20260925-002); multi-package libraries |
| 29 | Selective imports, re-exports, workspaces | none: qualified names only; local development by path replacements ([IMP-6], [MOD-4]) | verbosity; repeated replacements across local modules | experience with real multi-module projects |
| 30 | Standard library versioning | supplied by the toolchain at its version; manifests state a minimum language version and nothing is downloaded ([STD-1], [MAN-7]) | library changes tied to compiler releases | the first standard library |
| 31 | Letters in import paths | lowercase ASCII only ([PATH-2]) | origins with uppercase names need an explicit source | origins where lowercase paths cannot be obtained |
