# Decision 0001: Confirmed author decisions

- Status: Author — plans/decisions/README.md, L-0001 (confirmed by the author, Paulo R. Lima)
- Date: 2026-09-25
- Source: the author's bootstrap instructions for the first implementation cycle, §2

## Context

Before implementation started, the author fixed a set of decisions about the
identity and nature of the language. They are recorded here once, so that every
other document can link to them instead of restating them. They are not
reopened without the author's request.

## Decision

| Topic | Decision |
|---|---|
| Name | **Yxy**. |
| Domain | `yxy.dev`, owned by the author. Owning it does not authorize publishing a site or configuring DNS. |
| Source files | `.yxy` (for example `main.yxy`). Older extensions (`.ty`, `.la`, `.lina`) are not used in new code. |
| Nature | Open source language for the community, not a product built for sale. |
| Reach | Global. Its Brazilian origin is part of its identity, not a restriction on its audience. |
| Authorship in communications | Paulo R. Lima. No invented affiliations or rights holders. |
| Category | A language of its own: compiled, systems-level and general-purpose. Not an LLM, not an AI DSL, not a shell, not a framework. |
| References | Go, Rust, C and C++ are engineering mirrors; other languages may inform individual decisions. |
| Independence | Not a fork of Go, not a transpiler to Go, not an extension of Arandu, not dependent on the Go runtime. |
| Implementation constraint | The compiler is **not** implemented in Go. |
| Central pattern | Functions keep control on the left, the central production in the middle and propagation on the right: `@ctrl <- @eval -> @effect/@out`. |
| Technical audience | Humans and AI agents must be able to write, understand, review and modify Yxy code. |
| Priorities | Simplicity, correctness, resource control, predictability and performance, without undemonstrated promises. |

## Consequences

- Directory layout, command names, the bootstrap language and syntax details
  not listed above are **proposals** and are recorded as experimental decisions
  (decisions 0002 onward in this repository and in the compiler repository) until validated.
- Earlier reports that recommend a Go bootstrap, other names or
  commercialization do not revoke this record.
- `@io` is not used as a generic name for the output region in the new grammar;
  the output region is `@out`.
- Later decisions of the author (2026-09-25) are recorded in the compiler
  repository: Rust only as bootstrap scaffolding (implementation decision
  0001); private repositories, BSD 3-Clause license with **HYZIS - SERVICOS DIGITAIS LTDA - EPP** as
  copyright holder, commits authored by Paulo R. Lima (implementation decision
  0002).
