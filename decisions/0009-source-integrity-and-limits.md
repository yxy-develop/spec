# Decision 0009: Source integrity and limits

- Status: Experimental — plans/decisions/README.md, L-0009
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
5. Enums have between 1 and 256 variants in this version (the internal tag is
   one byte). Parameters follow the same naming rules as local variables.
6. An expression that never produces a value can only be a statement.

Items 5 and 6 come from an adversarial review of the compiler on 2026-09-25,
which found that a 257th variant compared equal to the first in native code,
and that a parameter named `None` silently changed what `None` meant.

## What could change it

Unicode identifiers (which would need confusable detection as well); real
programs that need deeper nesting; an explicit, traceable array copy operation.
