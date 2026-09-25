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
- **[LEX-5]** Keywords: `module enum struct fn extern export effects mut
  require else return if while match true false`.
- **[LEX-6]** Reserved words, rejected as "not supported yet": `as async await
  break const continue defer dyn for impl import in loop package par pub self
  static trait type unsafe use when where yield`. Packages, modules and imports
  are specified, not yet implemented, in `modules.md` (decision 0011); there
  `package`, `import`, `pub` and `as` become keywords (`modules.md` §9).
- **[LEX-7]** `let` and `var` are identifiers, but a statement that starts with
  `let name` or `var name` is rejected with a note pointing to `:=`.
- **[LEX-8]** Conventions: `snake_case` for functions, variables and modules;
  `PascalCase` for types and variants. They are documented, not enforced.

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
- **[LEX-11]** `1.5` is rejected as an unsupported floating-point literal.
  String (`"..."`) and character (`'x'`) literals are not supported yet.

### 1.4 Markers and punctuation

| Group | Tokens |
|---|---|
| Region markers | `@ctrl` `@eval` `@effect` `@out` |
| Region arrows | `<-` `->` |
| Typed hole | `$` or `$name` |
| Delimiters | `(` `)` `[` `]` `{` `}` |
| Separators | `,` `:` `.` `=>` |
| Declaration, assignment | `:=` `=` |
| Operators | `+ - * / % << >> & \| ^ == != < <= > >= && \|\| !` |
| Postfix | `?` |
| Array length | `;` (only inside `[T; N]` and `[value; N]`) |

- **[LEX-12]** Any other `@name` is an error. `@io` in particular is rejected,
  with a mechanical fix to `@out`.
- **[LEX-13]** Tokens are read with maximal munch. Consequently `<-` is always
  the region arrow: `a<-b` is an error; a comparison with a negative value is
  written `a < -b`.
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
  declarations and in literals), match arms and region headers, and inside the
  braces of `effects { ... }`. Inside the braces of a struct literal, `NL` is
  significant again even when the literal is inside `( )` or `[ ]`; it
  separates fields like `,`.
- **[NL-7]** A region header may be followed by a statement on the same line
  (compact form) or by statements on the following lines (multi-line form).
  A new region always starts on its own line.
- **[NL-8]** The `{` of `if`, `while` and `match` is on the same line as the
  end of the condition or subject. (A struct literal in that position is
  parenthesized: [GR-6].)

## 3. Grammar

```ebnf
module      = "module" IDENT NL { item } ;
item        = enum_decl | struct_decl | fn_decl ;

enum_decl   = "enum" IDENT "{" [ variant { ( "," | NL ) variant } [ "," ] ] "}" NL ;
variant     = IDENT ;

struct_decl = "struct" IDENT "{" [ field { ( "," | NL ) field } [ "," ] ] "}" NL ;
field       = IDENT ":" type ;

fn_decl     = [ "export" | "extern" ] "fn" IDENT "(" [ params ] ")" [ "->" type ]
              effects [ body ] NL ;
params      = param { "," param } [ "," ] ;
param       = IDENT ":" type ;
effects     = "effects" "{" [ IDENT { "," IDENT } [ "," ] ] "}" ;
body        = "{" ( cell | { stmt } ) "}" ;          (* required unless `extern` *)

cell        = region { region } ;
region      = region_head { stmt } ;
region_head = "@ctrl" ":"
            | "<-" "@eval" ":"
            | "->" "@effect" ":"
            | "->" "@out" ":" ;

block       = "{" { stmt } "}" ;
stmt        = ( decl | assign | require | return | if | while | expr ) ( NL | (* before "}" *) ) ;
decl        = [ "mut" ] IDENT [ ":" type ] ":=" expr
            | "_" ":=" expr ;
assign      = place "=" expr ;
place       = IDENT | IDENT "[" expr "]" | IDENT "." IDENT { "." IDENT } ;
require     = "require" expr "else" expr ;
return      = "return" [ expr ] ;
if          = "if" head block [ "else" ( block | if ) ] ;
while       = "while" head block ;
head        = expr ;              (* no unparenthesized struct literal: [GR-6] *)

type        = IDENT [ "<" type { "," type } ">" ]
            | "[" type ";" INT "]"
            | "&" "[" type "]"
            | "(" ")" ;

expr        = unary { binop unary } ;                (* see §4 *)
unary       = ( "-" | "!" | "&" ) unary | postfix ;
postfix     = primary { "(" [ expr { "," expr } [ "," ] ] ")"   (* primary is an IDENT *)
                      | "[" expr "]"
                      | "." IDENT
                      | "?" } ;
primary     = INT | "true" | "false" | IDENT | HOLE
            | "(" ")" | "(" expr ")"
            | "[" [ expr { "," expr } [ "," ] ] "]"
            | "[" expr ";" INT "]"
            | struct_lit
            | match ;
struct_lit  = IDENT "{" [ field_init { ( "," | NL ) field_init } [ "," ] ] "}" ;
field_init  = IDENT ":" expr ;
match       = "match" head "{" [ arm { ( "," | NL ) arm } [ "," ] ] "}" ;
arm         = pattern "=>" ( expr | block ) ;
pattern     = "_" | "true" | "false" | [ "-" ] INT | "(" ")"
            | IDENT                                  (* binding, or `None` *)
            | IDENT "(" pattern ")"                  (* Some, Ok, Err *)
            | IDENT "." IDENT ;                      (* Enum.Variant *)
```

- **[GR-1]** Only a name can be called: `f(x)`. Method calls (`x.f()`) are not
  supported.
- **[GR-2]** A cell occupies the whole function body. Region headers after
  ordinary statements, or inside nested blocks, are errors.
- **[GR-3]** `if` is a statement, not an expression. `match` is an expression.
- **[GR-4]** Grouping parentheses carry no meaning beyond grouping.
- **[GR-5]** Limits: the **combined** nesting depth of expressions, blocks,
  `else if` links, types and patterns is at most 256 levels; at one level of
  parentheses an expression chains at most 4096 binary operators, and at most
  4096 postfix operators. Deeper input is rejected with a diagnostic instead of
  exhausting the compiler's resources.
- **[GR-6]** *(experimental)* `IDENT "{"` starts a struct literal, except in a
  `head`: the condition of `if` (also after `else`) and `while`, and the
  subject of `match`, outside any `( )`, `[ ]` or `{ }` nested in it. There the
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
  struct expressions; see `decisions/0012-structs-and-layout.md`.
- **[GR-7]** `.` followed by a name is a field access (`p.x`, `a.b.c`), the
  length of an array or slice (`s.len`), or a variant (`Enum.Variant`);
  which one is decided by what precedes the `.` (`semantics.md` §2.1).
  Struct literals have no field shorthand (`Point { x, y }`) and no update
  syntax (`Point { x: 1, ..p }`).

## 4. Operators

| Precedence (high → low) | Operators | Associativity |
|---|---|---|
| 11 | postfix: call, `[i]`, `.name` (field, `.len`, variant), `?` | left |
| 10 | prefix: `-` `!` `&` | right |
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
`(a & mask) == 0`.

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
