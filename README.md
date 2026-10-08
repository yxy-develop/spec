<p align="center">
  <img src=".github/logo.svg" alt="Yxy" width="180">
</p>

<h1 align="center">yxy-develop/spec</h1>

<p align="center">The Yxy language specification: how the language should behave, and why.</p>

<p align="center">
<a href="https://yxy.dev">yxy.dev</a> ·
<a href="https://yxy.dev/docs">Documentation</a> ·
<a href="https://github.com/yxy-develop/yxy">Compiler</a> ·
<a href="https://github.com/orgs/yxy-develop/discussions">Discussions</a>
</p>

## About the specification

This repository defines **how the Yxy language should behave**. It is
experimental and changes through recorded decisions until 1.0.

| File | Content |
|---|---|
| `syntax.md` | Lexical structure, newline rules, grammar (EBNF), operators |
| `semantics.md` | Types, structs, enums with data, declarations, constants, borrows, borrowed text, ownership (moves and the explicit copy), cells, `match` and its patterns, loops, evaluation, integer rules, floating point, effects, the console capability, the C boundary, a glossary |
| `modules.md` | Packages, modules, imports, visibility, manifest and lock — experimental; packages of one module, dependencies from directories and Git origins, the lock and fetching implemented; the package service not yet |
| `decisions/NNNN-*.md` | Language decisions, with their state and register ID, context, alternatives and what could change them |
| `decisions/OPEN.md` | Open questions with the provisional choice in force |

The specification describes intent. What the compiler implements today is
tracked separately, requirement by requirement, in the compiler repository
([`yxy-develop/yxy`](https://github.com/yxy-develop/yxy),
[`docs/implementation/STATUS.md`](https://github.com/yxy-develop/yxy/blob/HEAD/docs/implementation/STATUS.md)).
A divergence between the two is recorded there and resolved by a decision,
never by silently editing this text to match the code.

Rule identifiers such as `[CELL-4]` are stable anchors for tests and
discussions.

To learn the language, start with the [guide](https://yxy.dev/docs/guide) on
the site; this text is the reference it answers to.

## Proposing a change

A change to the language starts as an RFC in
[Yxy Discussions](https://github.com/orgs/yxy-develop/discussions)
(RFC / Proposals) and becomes a file in `decisions/` (context, decision,
alternatives, what could change it) before the text of `syntax.md`,
`semantics.md` or `modules.md` changes. The
[contribution guide](https://github.com/yxy-develop/.github/blob/HEAD/CONTRIBUTING.md),
[support policy](https://github.com/yxy-develop/.github/blob/HEAD/SUPPORT.md),
[security policy](https://github.com/yxy-develop/.github/blob/HEAD/SECURITY.md)
and [code of conduct](https://github.com/yxy-develop/.github/blob/HEAD/CODE_OF_CONDUCT.md)
are shared by every Yxy repository. Anything else:
[contact@yxy.dev](mailto:contact@yxy.dev).

## Em português

Este repositório define **como a linguagem Yxy deve se comportar**. É
experimental e muda por decisões registradas até a versão 1.0.

- `syntax.md`: léxico, regras de quebra de linha, gramática e operadores.
- `semantics.md`: tipos, structs, enums com dados, declarações, constantes,
  empréstimos, texto emprestado, posse (moves e a cópia explícita), células (`@ctrl <- @eval -> @effect/@out`),
  `match` e seus padrões, laços, avaliação, aritmética inteira, ponto
  flutuante, efeitos, a capacidade de console, fronteira com C e um glossário.
- `modules.md`: pacotes, módulos, imports, visibilidade, manifesto e lock
  (experimental; pacotes de um módulo, dependências de diretórios e de
  origens Git, o lock e o `fetch` já implementados; o serviço de pacotes
  ainda não).
- `decisions/`: decisões de linguagem, com estado e ID do registro, e questões abertas.

A especificação descreve a intenção. O que o compilador implementa hoje fica
registrado, requisito por requisito, no repositório do compilador
(`yxy-develop/yxy`, `docs/implementation/STATUS.md`). Para aprender a
linguagem, comece pelo [guia em português](https://yxy.dev/pt/docs/guia).
Uma mudança na linguagem começa como RFC nas
[Discussões da Yxy](https://github.com/orgs/yxy-develop/discussions) e vira
uma decisão em `decisions/` antes de mudar o texto. Contato:
[contact@yxy.dev](mailto:contact@yxy.dev).

## License

BSD 3-Clause, copyright HYZIS - SERVICOS DIGITAIS LTDA - EPP. See
[`LICENSE`](LICENSE).
