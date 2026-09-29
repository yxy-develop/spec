# Yxy language specification

This repository defines **how the Yxy language should behave**. It is
experimental and changes through recorded decisions until 1.0.

| File | Content |
|---|---|
| `syntax.md` | Lexical structure, newline rules, grammar (EBNF), operators |
| `semantics.md` | Types, structs, enums with data, declarations, constants, borrows, borrowed text, cells, `match` and its patterns, loops, evaluation, integer rules, floating point, effects, the console capability, the C boundary, a glossary |
| `modules.md` | Packages, modules, imports, visibility, manifest and lock — experimental; packages of one module, dependencies from directories and Git origins, the lock and fetching implemented; the package service not yet |
| `decisions/NNNN-*.md` | Language decisions, with their state and register ID, context, alternatives and what could change them |
| `decisions/OPEN.md` | Open questions with the provisional choice in force |

The specification describes intent. What the compiler implements today is
tracked separately, requirement by requirement, in the compiler repository
(`yxy-develop/yxy`, `docs/implementation/STATUS.md`). A divergence between the
two is recorded there and resolved by a decision, never by silently editing
this text to match the code.

Rule identifiers such as `[CELL-4]` are stable anchors for tests and
discussions.

## Em português

Este repositório define **como a linguagem Yxy deve se comportar**. É
experimental e muda por decisões registradas até a versão 1.0.

- `syntax.md`: léxico, regras de quebra de linha, gramática e operadores.
- `semantics.md`: tipos, structs, enums com dados, declarações, constantes,
  empréstimos, texto emprestado, células (`@ctrl <- @eval -> @effect/@out`),
  `match` e seus padrões, laços, avaliação, aritmética inteira, ponto
  flutuante, efeitos, a capacidade de console, fronteira com C e um glossário.
- `modules.md`: pacotes, módulos, imports, visibilidade, manifesto e lock
  (experimental; pacotes de um módulo, dependências de diretórios e de
  origens Git, o lock e o `fetch` já implementados; o serviço de pacotes
  ainda não).
- `decisions/`: decisões de linguagem, com estado e ID do registro, e questões abertas.

A especificação descreve a intenção. O que o compilador implementa hoje fica
registrado, requisito por requisito, no repositório do compilador
(`yxy-develop/yxy`, `docs/implementation/STATUS.md`).

## License

BSD 3-Clause. See `LICENSE`.
