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
