# Yxy syntax — experimental, slice 1

Status: **experimental**. This document is normative for the first implemented
subset ("slice 1"). Everything here may change before 1.0 through a recorded
decision in `decisions/`. Rule identifiers such as `[LEX-3]` are stable
anchors for tests and discussions.

Notation: EBNF with `{ x }` = zero or more, `[ x ]` = optional, `|` =
alternative, `"x"` = literal token. `NL` is the newline token (see §2).

## 1. Lexical structure

### 1.1 Source text

- **[LEX-1]** A source file is UTF-8 text with the extension `.yxy`. A UTF-8
  byte order mark at the start is ignored.
- **[LEX-2]** Space, horizontal tab and carriage return are whitespace. A line
  feed ends a line. Indentation has no meaning.
- **[LEX-3]** Comments start with `//` and run to the end of the line. They may
  contain any UTF-8 text except the characters of [LEX-3a]. Block comments
  (`/* */`) are not part of the language.
- **[LEX-3a]** These characters are errors **anywhere** in a file, comments
  included, so that code always reads the way it compiles ("Trojan Source"):
  control characters other than tab, line feed and a carriage return followed
  by a line feed; DEL and the C1 controls U+0080–U+009F; U+2028; U+2029; the
  bidirectional formatting
  characters U+061C, U+200E, U+200F, U+202A–U+202E and U+2066–U+2069; and
  U+FEFF anywhere but at the start.

### 1.2 Identifiers and words

- **[LEX-4]** `IDENT = [A-Za-z_][A-Za-z0-9_]*`. Identifiers are ASCII in this
  version; a letter outside ASCII in an identifier is an error. A lone `_` is
  not an identifier: it is the wildcard/discard token.
- **[LEX-5]** Keywords: `package import pub as enum struct fn extern export
  effects mut require else return if while loop for in break continue match
  true false const unsafe`. *(`loop`, `for`, `in`, `break` and `continue`
  since decision 0016, `const` since decision 0018, and *(experimental)*
  `unsafe` since decision 0024; they were reserved words before.)*
- **[LEX-6]** Reserved words, rejected as "not supported yet": `async await
  defer dyn impl let module par self static trait type use var when
  where yield`. `module` is rejected with a mechanical fix
  to `package` at the start of a file (`modules.md` [MIG-1]); `use` stays
  reserved (`modules.md` [IMP-6]); `let` and `var` are rejected with a note
  pointing to `:=` instead ([LEX-7]). The compiler's conformance suite copies the
  lists of [LEX-5] and [LEX-6] word for word and compares them with the
  lexer's: a change to either list is made in the specification, in the lexer
  and in that test together.
- **[LEX-7]** *(decision 0002, amendment of 2026-10-04)* `let` and `var` are
  reserved words ([LEX-6]), never names: using either as a name is an error
  with a note pointing to `:=`, and a statement that starts with `let name`
  or `var name` is rejected with the same note and, when the rest is a
  declaration, a mechanical fix to `:=` (`let x: u8 = 1` becomes
  `x: u8 := 1`, `var` becomes `mut`). *(Before the amendment they were
  identifiers, and only such a statement was rejected.)*
- **[LEX-8]** Conventions: `snake_case` for functions, variables and packages;
  `PascalCase` for types and variants. They are documented, not enforced.
  Import path elements are lowercase, which is enforced (`modules.md`
  [PATH-2]).

### 1.3 Literals

```ebnf
INT     = dec | hex | bin ;
dec     = "0" | nonzero { [ "_" ] digit } ;
hex     = "0x" hexdigit { [ "_" ] hexdigit } ;
bin     = "0b" bindigit { [ "_" ] bindigit } ;
```

- **[LEX-9]** A single `_` separates two digits (`1_000`; not `1__000`, `1_`,
  `0x_1`). Leading zeros (`007`), type suffixes
  (`10u8`) and values above 2^128 − 1 are errors. There is no octal.
- **[LEX-10]** An integer literal has no type of its own: its type comes from
  context (see `semantics.md` [TY-4]).
- **[LEX-11]** *(decision 0019)* `1.5` is a float literal ([LEX-19]).
  Character literals (`'x'`) are not supported: a code point is not a type in
  this version, and text of one code point is a string literal, `"x"`. A
  double-quoted literal is the path of an import right after `import`
  ([LEX-17]), and a string literal anywhere else ([LEX-18]).
  *(Before decision 0015 a string literal was refused outside an import;
  before decision 0019 a float literal was refused.)*
- **[LEX-17]** The double-quoted literal right after `import` is the path of
  the import (`modules.md` [IMP-1]): on one line, without escape sequences,
  printable ASCII only; its content is checked by `modules.md`
  [PATH-1]–[PATH-5].
- **[LEX-18]** *(experimental, decision 0015)* Anywhere else a double-quoted
  literal is a **string literal**, whose value is text (`semantics.md`
  [TEXT-1]). It closes on the line where it starts. Inside it, only printable
  ASCII (U+0020–U+007E) other than `"` and `\` stands for itself; everything
  else is written as an escape:

  | Escape | Value |
  |---|---|
  | `\n` `\t` `\r` `\0` | line feed, tab, carriage return, NUL |
  | `\\` `\"` | `\` and `"` |
  | `\u{H…}` | the code point of 1 to 6 hexadecimal digits (either case), which must be a Unicode scalar value: not a surrogate (U+D800–U+DFFF), at most U+10FFFF |

  Any other escape, a malformed `\u{…}`, a character outside printable ASCII
  written as itself (a non-ASCII letter, a tab) and a literal that does not
  close on its line are errors; for a character written as itself the
  mechanical fix is its escape, with the same bytes (`ç` becomes `\u{E7}`).
  The characters of [LEX-3a] are errors here as everywhere. The value of the
  literal is the UTF-8 encoding of its code points, escapes decoded, so it is
  always valid UTF-8. Writing non-ASCII text through escapes keeps the bytes
  a reader sees equal to the bytes the program holds, whatever the
  normalization or look-alike characters of an editor; allowing UTF-8 written
  as itself later would only accept more programs.

- **[LEX-19]** *(experimental, decision 0019)* A **float literal** is decimal
  digits, `.`, decimal digits, and an optional exponent `e` or `e-` followed
  by decimal digits:

  ```ebnf
  FLOAT  = dec "." digits [ "e" [ "-" ] dec ] ;
  digits = digit { [ "_" ] digit } ;
  ```

  `dec` is the decimal integer of [LEX-9] (no leading zeros), and `_`
  separates two digits as in integers (`1_000.000_1`). Each form has one
  spelling (a value can still be written in several forms: `1.5`, `1.50`,
  `15.0e-1`): `1.` (no digits after `.`), `1e5` (an exponent without the
  fraction), `1.5E3` (a capital `E`), `1.5e+3` (a `+`) and `1.0e05`
  (leading zeros in the exponent) are errors, each with a mechanical fix to
  the one spelling of its form (`1.0`, `1.0e5`, `1.5e3`, `1.5e3`,
  `1.0e5`); `.5` is read as `.` followed by `5` and so is refused as a
  malformed expression. A type suffix (`1.5f32`), leading zeros (`007.5`),
  a misplaced `_` and an exponent without digits are errors without a fix.
  There are no hexadecimal float literals and no literals for infinities or
  NaN. `1.len` and `1..2` are an integer followed by `.`, not a float. A
  float literal has no type of its own: its type comes from context
  (`semantics.md` [TY-4], [FLT-2]).

```ebnf
PATH      = '"' path_char { path_char } '"' ;
path_char = (* printable ASCII other than '"'; the path is then checked by modules.md [PATH-1]–[PATH-5] *) ;
STRING    = '"' { text_char | escape } '"' ;          (* [LEX-18]; one line *)
text_char = (* U+0020–U+007E other than '"' and '\' *) ;
escape    = '\' ( "n" | "t" | "r" | "0" | '\' | '"' ) | '\u{' hexdigit [ hexdigit ] [ hexdigit ] [ hexdigit ] [ hexdigit ] [ hexdigit ] "}" ;
```

### 1.4 Markers and punctuation

| Group | Tokens |
|---|---|
| Region markers | `@ctrl` `@eval` `@effect` `@out` |
| Region arrows | `<-` `->` |
| Typed hole | `$` or `$name` |
| Delimiters | `(` `)` `[` `]` `{` `}` |
| Separators | `,` `:` `.` `=>` |
| Range | `..` `..=` (only in the head of `for`, [GR-8]) |
| Declaration, assignment | `:=` `=` |
| Operators | `+ - * / % << >> & \| ^ == != < <= > >= && \|\| !` |
| Postfix | `?` |
| Array length | `;` (only inside `[T; N]` and `[value; N]`) |

- **[LEX-12]** Any other `@name` is an error. `@io` in particular is rejected,
  with a mechanical fix to `@out`.
- **[LEX-13]** Tokens are read with maximal munch. Consequently `<-` is always
  the region arrow: `a<-b` is an error; a comparison with a negative value is
  written `a < -b`. No other valid program is affected: `a>-b`, `a>=-b`,
  `a<=-b` and `a--b` read as an operator followed by a unary minus, and
  reading `->` or `=>` as two tokens could not give a valid program, because
  `>` is not a prefix operator.
- **[LEX-14]** `->` has two roles decided by position: after the parameter list
  it introduces the return type; at the start of a region header it is the
  region arrow.
- **[LEX-15]** In a type argument list, `>>` closes two lists
  (`Option<Option<u8>>`).
- **[LEX-16]** `;` is not a statement terminator.

## 2. Lines and statement termination

- **[NL-1]** The lexer produces one `NL` token for each line break (consecutive
  blank lines and comment-only lines collapse), **except** while the innermost
  open delimiter is `(` or `[`: parameter lists, argument lists and array
  literals may span lines freely.
- **[NL-2]** A statement ends at `NL` or just before the `}` that closes its
  block.
- **[NL-3]** A line that ends with a binary operator continues on the next line.
  A line that *starts* with an operator does **not** continue the previous line.
- **[NL-4]** `else` may start the line after `require cond` and after the `}` of
  an `if`.
- **[NL-5]** The parts of a function signature may be on separate lines:
  `)`, `-> Type`, `effects { ... }` and the body `{`.
- **[NL-6]** `NL` is ignored around items, enum variants, struct fields (in
  declarations, literals and patterns), match arms and region headers, and
  inside the braces of `effects { ... }`. Inside the braces of a struct
  literal or a struct pattern, `NL` is significant again even when it is
  inside `( )` or `[ ]`; it separates fields like `,`. The package clause and each import end at their
  line break (`modules.md` §9).
- **[NL-7]** A region header may be followed by a statement on the same line
  (compact form) or by statements on the following lines (multi-line form).
  A new region always starts on its own line.
- **[NL-8]** The `{` of `if`, `while`, `for`, `loop` and `match` is on the
  same line as the end of the condition, head or subject (for `loop`, as the
  keyword). (A struct literal in that position is parenthesized: [GR-6].)

## 3. Grammar

```ebnf
file        = package_clause { import_decl } { item } ;   (* modules.md §9 *)
package_clause
            = "package" IDENT NL ;
import_decl = "import" PATH [ "as" IDENT ] NL ;
item        = [ "pub" ] ( enum_decl | struct_decl | fn_decl | const_decl ) ;

qual_name   = IDENT [ "." IDENT ] ;             (* item, or import.item *)

enum_decl   = "enum" IDENT "{" [ variant { ( "," | NL ) variant } [ "," ] ] "}" NL ;
variant     = IDENT [ "(" type { "," type } [ "," ] ")" ] ;    (* data: semantics.md [ENUM-1] *)

struct_decl = "struct" IDENT "{" [ field { ( "," | NL ) field } [ "," ] ] "}" NL ;
field       = [ "pub" ] IDENT ":" type ;          (* modules.md [VIS-2], OPEN #28 *)

fn_decl     = [ "export" | "extern" | "drop" ] "fn" IDENT "(" [ params ] ")" [ "->" type ]
              effects [ body ] NL ;   (* `drop`: decision 0021, a word only before `fn`, semantics.md [DROP-1] *)
const_decl  = "const" IDENT ":" type ":=" expr NL ;   (* decision 0018; semantics.md [CONST-1], [CONST-2] *)
params      = param { "," param } [ "," ] ;
param       = [ "take" | "inout" ] IDENT ":" type ;   (* decision 0020: words only before a name, semantics.md [OWN-3], [OWN-9] *)
effects     = "effects" "{" [ IDENT { "," IDENT } [ "," ] ] "}" ;
body        = "{" ( cell | { stmt } ) "}" ;          (* required unless `extern` *)

cell        = region { region } ;
region      = region_head { stmt } ;
region_head = "@ctrl" ":"
            | "<-" "@eval" ":"
            | "->" "@effect" ":"
            | "->" "@out" ":" ;

block       = "{" { stmt } "}" ;
stmt        = ( decl | assign | require | return | if | while | loop | for
              | "break" | "continue" | expr ) ( NL | (* before "}" *) ) ;
decl        = [ "mut" ] IDENT [ ":" type ] ":=" expr
            | "_" ":=" expr ;
assign      = place "=" expr ;
place       = IDENT | IDENT "[" expr "]" | IDENT "." IDENT { "." IDENT } ;
require     = "require" expr "else" expr ;
return      = "return" [ expr ] ;
if          = "if" head block [ "else" ( block | if ) ] ;
while       = "while" head block ;
loop        = "loop" block ;                                  (* decision 0016 *)
for         = "for" ( IDENT | "_" ) [ ":" type ] "in" for_source block ;
for_source  = head ( ".." | "..=" ) head                      (* [GR-8] *)
            | head ;
head        = expr ;              (* no unparenthesized struct literal: [GR-6] *)

type        = qual_name [ "<" type { "," type } ">" ]
            | "[" type ";" expr "]"                 (* the length: a constant expression, [CONST-5] *)
            | "&" "[" type "]"
            | "&" "mut" "[" type "]"                (* decision 0020, TASK-042 part: semantics.md [REF-6] *)
            | "(" ")" ;

expr        = unary { binop unary } ;                (* see §4 *)
unary       = ( "-" | "!" | "&" | "&" "mut" ) unary | postfix ;   (* `&mut`: [REF-6] *)
postfix     = primary { "(" [ arg { "," arg } [ "," ] ] ")"   (* callee is a name, import.name, Enum.Variant, import.Enum.Variant, console.operation, x.copy (decision 0020), Mmio.at or handle.operation (decision 0024) *)
                      | "[" expr "]"
                      | "." IDENT
                      | "?" } ;
arg         = [ "inout" ] expr ;   (* decision 0020, TASK-042 part: `inout` is a word only before an operand that starts with IDENT, semantics.md [OWN-9] *)
primary     = INT | FLOAT | STRING | "true" | "false" | IDENT | HOLE
            | "(" ")" | "(" expr ")"
            | "[" [ expr { "," expr } [ "," ] ] "]"
            | "[" expr ";" expr "]"                 (* the count: a constant expression, [CONST-5] *)
            | struct_lit
            | match
            | unsafe ;
unsafe      = "unsafe" STRING "{" [ NL ] expr [ NL ] "}" ;   (* decision 0024: an unsafe region, semantics.md [UNS-1] *)
struct_lit  = qual_name "{" [ field_init { ( "," | NL ) field_init } [ "," ] ] "}" ;
field_init  = IDENT ":" expr ;
match       = "match" head "{" [ arm { ( "," | NL ) arm } [ "," ] ] "}" ;
arm         = pattern "=>" ( expr | block ) ;
pattern     = "_" | "true" | "false" | [ "-" ] INT | "(" ")"
            | [ "-" ] FLOAT                          (* parsed, always an error: [FLT-10] *)
            | IDENT                                  (* binding, or `None` *)
            | IDENT "(" patterns ")"                 (* Some, Ok, Err *)
            | IDENT "." IDENT [ "(" patterns ")" ]   (* Enum.Variant, with its values *)
            | IDENT "." IDENT "." IDENT [ "(" patterns ")" ]   (* import.Enum.Variant *)
            | qual_name "{" [ field_pat { ( "," | NL ) field_pat } [ "," ] ] "}" ;   (* [GR-9] *)
patterns    = pattern { "," pattern } [ "," ] ;
field_pat   = IDENT ":" pattern ;
```

A float literal is parsed as a pattern only to be refused: floats are matched
by `_` or a binding (`semantics.md` [FLT-10], decision 0019). Patterns have
no other forms in this version: or-patterns (`p | q`), guards
(`pattern if condition`) and ranges (`a..b`) are open (`decisions/OPEN.md`
#46), and an index is one expression, never a range (`s[i..j]`, #47). Each is
rejected with a diagnostic of its own (`semantics.md` §12), also when the
pattern goes on over lines (`p` on one line, `| q` or `if condition` on the
next). A value that is not used is written `_` (`Some(_)`,
`Shape.Rect(w, _)`, `Point { x: 0, y: _ }`): there are no rest patterns
(`Some(..)`, `Shape.Rect(..)`, `Point { x: 0, .. }`) and no field shorthand
in struct patterns (`Point { x }`), and a variant is written with one `.`
(`Color.Red`). *(decision 0017)* Variants with their values and struct
patterns are part of the grammar (`semantics.md` [MATCH-4], [MATCH-5]);
before decision 0017 they were refused (#33).

- **[GR-1]** Only a name or a qualified name whose first part is an import can
  be called: `f(x)`, `pkg.f(x)` (`modules.md` [IMP-6]). *(experimental,
  decision 0015)* The one exception is an operation of a `Console`
  capability, written with the capability before `.`: `console.print(x)`,
  where `console` is a parameter or local variable of type `Console`
  (`semantics.md` [CON-2]); names of imports and of variables never collide
  ([IMP-5]), so the first name says which form it is. *(experimental,
  decision 0020)* The second exception is the explicit copy, `.copy()`
  with no argument after any operand but an import's name: `p.copy()`,
  `p.inner.copy()`, `a[i].copy()`, `make().copy()` (`semantics.md`
  [OWN-6]); `pkg.copy()` stays a call of the function `copy` of the import
  `pkg`. Other method calls (`x.f()`) are not supported.
- **[GR-2]** A cell occupies the whole function body. Region headers after
  ordinary statements, or inside nested blocks, are errors.
- **[GR-3]** *(experimental)* `if` is a statement, not an expression.
  `match` is an expression: a value chosen by a condition is a `match` on it
  (`match c { true => a, false => b }`). An `if` where a value is expected is
  rejected with the first sentence of this rule and a note pointing to
  `match`. Making `if` an expression later would change no valid program,
  since none has an `if` where a value is expected; making `match` a
  statement would, and once the language is open that needs the evolution
  rule of `decisions/OPEN.md` #48.
- **[GR-4]** Grouping parentheses carry no meaning beyond grouping.
- **[GR-5]** Limits: the **combined** nesting depth of expressions, blocks,
  `else if` links, types and patterns is at most 256 levels; at one level of
  parentheses an expression chains at most 4096 binary operators, and at most
  4096 postfix operators. Deeper input is rejected with a diagnostic instead of
  exhausting the compiler's resources.
- **[GR-6]** *(experimental)* `IDENT "{"` (or `IDENT "." IDENT "{"`, a struct
  of an imported package) starts a struct literal, except in a
  `head`: the condition of `if` (also after `else`) and `while`, the subject
  of `match` and the source of `for` (each bound of a range), outside any
  `( )`, `[ ]` or `{ }` nested in it. There the
  `{` belongs to the statement (its block, or the arms of `match`), so a struct
  literal is written in parentheses:

  ```yxy
  if (Point { x: 0, y: 0 }).x == p.x {         // valid
  if distance(Point { x: 0, y: 0 }, p) > 3 {   // valid: inside a call's ( )
  if Point { x: 0, y: 0 }.x == p.x {           // error: needs parentheses
  ```

  When an unparenthesized struct literal in a head is followed by an operator,
  `.`, `?`, `[` or `{`, the compiler reports that the literal needs
  parentheses (and offers them as a mechanical fix) instead of reporting a
  malformed block. `yxy fmt` always writes a struct literal in a head inside
  parentheses. The rule is Go's rule for composite literals and Rust's for
  struct expressions; see `decisions/0012-structs-and-layout.md`. A literal
  in a head that is nested inside another such literal (in the subject of a
  `match` within a field value) is reported too; the fix of the outer literal
  already includes the inner one.
- **[GR-7]** `.` followed by a name is a field access (`p.x`, `a.b.c`), the
  length of an array or slice (`s.len`), the length in bytes or the byte view
  of text (`t.len`, `t.bytes`, `semantics.md` [TEXT-3], [TEXT-4]), or a
  variant (`Enum.Variant`); which one is decided by what precedes the `.`
  (`semantics.md` §2.1).
  Struct literals have no field shorthand (`Point { x, y }`) and no update
  syntax (`Point { x: 1, ..p }`).
- **[GR-8]** *(experimental, decision 0016)* `..` and `..=` separate the two
  bounds of a range in the head of `for`, and appear nowhere else: a range
  is not a value (`r := 0..3` is an error), a pattern (OPEN #46) or an index
  (OPEN #47). Each bound is a whole expression, so `..` binds more loosely
  than every operator: `for i in 0..n + 1` ends at `n + 1`. The two dots
  touch, and so does the `=` of `..=`. A range has both bounds: `a..` and
  `..b` are errors. `break` and `continue` are statements with nothing after
  them (no value, no label); in a `match` arm they are written in a block,
  `None => { break }`.
- **[GR-9]** *(experimental, decision 0017)* In a pattern, `Name {` (or
  `pkg.Name {`) starts a struct pattern. Inside the parentheses of a
  constructor or the braces of another struct pattern it always does. At the
  top of a `match` arm, where `{` after a pattern would otherwise be the block
  of an arm whose `=>` is missing, it does when field patterns follow up to a
  `}` (`field: pattern` or `field`, separated by `,` or line breaks, a `..`
  last) and the arm goes on after it (`=>`, or the `->`, `|` or `if` of
  other languages, each reported as such); otherwise the `{` is reported as
  a missing `=>`. Field patterns are separated by `,` or line
  breaks, as the fields of a literal ([NL-6]). A struct pattern appears only
  in a pattern: `Point { x: a, y: b } := p` is not a declaration.
- **[GR-10]** *(experimental, decision 0024)* `unsafe` is followed by a
  string literal, the reason, and by braces that hold one expression, which
  may stand on its own line between them: `unsafe "UART0 registers" {
  Mmio.at(UART0, 0x40) }`. It is a primary expression. A missing or empty
  reason (only spaces) is an error (E0620 of the compiler), and so are
  braces that hold nothing, more than one expression or a statement (E0621);
  the meaning is in `semantics.md` [UNS-1].

## 4. Operators

| Precedence (high → low) | Operators | Associativity |
|---|---|---|
| 11 | postfix: call, `[i]`, `.name` (field, `.len`, `.bytes`, variant), `.copy()` (decision 0020), `?` | left |
| 10 | prefix: `-` `!` `&` `&mut` (decision 0020, TASK-042 part) | right |
| 9 | `*` `/` `%` | left |
| 8 | `+` `-` | left |
| 7 | `<<` `>>` | left |
| 6 | `&` | left |
| 5 | `^` | left |
| 4 | `\|` | left |
| 3 | `==` `!=` `<` `<=` `>` `>=` | **none**: `a < b < c` is an error |
| 2 | `&&` | left |
| 1 | `\|\|` | left |

Bitwise operators bind tighter than comparisons, so `a & mask == 0` means
`(a & mask) == 0`. `..` and `..=` are not operators: they separate the bounds
of a range in the head of `for`, below every operator ([GR-8]).

## 5. The two presentations of a cell

The multi-line and the compact presentation are the same grammar and produce
the same syntax tree ([NL-7]):

```yxy
fn read_byte(
    data: &[u8],
    index: usize
) -> Result<u8, BufferError>
effects {}
{
    @ctrl:
        require index < data.len
            else BufferError.OutOfBounds

    <- @eval:
        value := data[index]

    -> @out:
        return Ok(value)
}
```

```yxy
fn read_byte(data: &[u8], index: usize) -> Result<u8, BufferError>
effects {}
{
    @ctrl: require index < data.len else BufferError.OutOfBounds
        <- @eval: value := data[index]
        -> @out: return Ok(value)
}
```

A function without regions needs no artificial cell:

```yxy
fn identity(value: u64) -> u64
effects {}
{
    return value
}
```

The canonical layout is the one produced by `yxy fmt`; it currently always
writes cells in the multi-line form.
