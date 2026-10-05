# Decision 0027: Documentation comments

- Status: Experimental — not accepted. Written under TASK-20260926-051 of
  the master plan (Phase 8: "`yxy doc`; consultas `context`/`impact` pela
  CLI sobre fatos versionados"). The orchestrator's choice, with the
  alternatives below, for the author's review; its state is controlled in
  the register of the control repository.
- Date: 2026-10-05
- Origin: TASK-20260926-051, whose `yxy doc` documents "the public items and
  their doc comments": the language had none ([LEX-3] has only `//`
  comments, which mean nothing).
- Spec: `syntax.md` [LEX-3b] (new). Nothing else changes: a documentation
  comment is a comment ([LEX-3], [LEX-3a]), and a program means what it meant.
- Implementation: implementation decision 0017 of the compiler (`yxy doc`).

## Context

A tool that documents a package needs to know which text documents which
item. Comments are free text that the compiler skips ([LEX-3]); a comment
before an item is often a note for whoever edits the code ("Not documented:
private", "TODO"), not documentation for whoever uses it. The language
needs a marker, as small as possible, that a reader and a tool both see.

## Decision

1. **[LEX-3b] A documentation comment** is a comment ([LEX-3]) whose text
   after `//` starts with `/` and not with `//`: `/// text`, alone on its
   line (only whitespace before it). `//` and `////…` comments, and a `///`
   after code on its line, are ordinary comments.
2. **What it documents.** A run of documentation comments, each on the line
   after the previous one, documents what starts on the line after the last
   one: an item (the grammar's `item`, from `pub` or its first keyword), a field of a
   struct, a variant of an enum, or the package clause (the package; a
   package of several files has the documentation of each file's clause, in
   the order of the files' names). A blank line, or anything but whitespace,
   between the run and what follows ends it: the run then documents nothing.
   Elsewhere (before an import, inside a body) a documentation comment is an
   ordinary comment.
3. **Its text** is each comment's text after `///` without one space that
   follows it, the lines joined by line feeds. It is Markdown for people:
   paragraphs separated by blank lines, `` `code` `` and fenced code blocks
   (```` ``` ````). HTML in it is text, never markup: a tool shows a tag as
   written and never interprets it, since the text of a program is data,
   never instructions (a tool that renders HTML renders only the three forms
   above, and escapes every other character).
4. **No meaning.** A documentation comment changes nothing of the program:
   no check, no code, no fact but the documentation; [LEX-3a] holds in it
   as in every comment. The formatter keeps it where it is, as it keeps
   every comment.

## Alternatives

- **Any comment right before an item is its documentation** (no marker):
  no new rule, but notes for whoever edits the code become documentation,
  and there is no way to write a comment before an item that is not one.
  Not taken.
- **An attribute or a keyword** (`doc "…"`, `@doc`): a construct of the
  grammar that the checker would have to check (nothing the parser accepts
  is ignored), for text that has no meaning. Not taken: a comment
  stays a comment.
- **`//!` for the package**, as Rust's inner comments: a second marker for
  one case; the comment before the package clause says the same with the one
  marker. Not taken; it can be added if packages need documentation that is
  not before a clause.
- **Blank lines allowed between the run and the item**: more forgiving, but a
  `///` comment left above a removed item would silently document the next
  one. Not taken.

## What could change it

A rule on links inside documentation text (to other items by ID), on
documentation of parameters, or a warning for a documentation comment that
documents nothing, when a tool needs them; the author's review.
