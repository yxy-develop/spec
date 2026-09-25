# Decision 0009: Source integrity and limits

- Status: Accepted — experimental
- Date: 2026-09-25
- Spec: `syntax.md` [LEX-3a], [GR-5]; `semantics.md` [TY-6], [DECL-6]
- Origin: adversarial review of the specification draft (2026-09-25), whose
  findings also applied to the implementation

## Decision

1. Control characters (other than tab and line breaks), lone carriage returns,
   invisible line/paragraph separators and bidirectional formatting characters
   are rejected **everywhere, comments included**. A reviewer — human or agent —
   must see the code the compiler compiles ("Trojan Source", CVE-2021-42574).
2. Nesting of expressions, blocks, `else if` links, types and patterns is
   limited to 256 levels, and one expression to 4096 chained operators.
   Pathological input yields a diagnostic, never a crash of the compiler.
3. Arrays are created only from literals; copying a whole array is rejected
   until copies can be made explicit and traceable.
4. A bare name in a pattern that equals a variant of the matched enum is an
   error instead of a binding that silently matches everything.

## What could change it

Unicode identifiers (which would need confusable detection as well); real
programs that need deeper nesting; an explicit, traceable array copy operation.
