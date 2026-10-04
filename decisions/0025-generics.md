# Decision 0025: Generics, static dispatch, function values and their effects

- Status: Experimental — not accepted. Written under step 1 of
  TASK-20260926-047 of the master plan (Phase 8), for the author's review;
  step 2 implements it (part 11). Every choice below is the orchestrator's,
  with the alternatives that follow; none is the author's.
- Date: 2026-10-04
- Origin: `decisions/OPEN.md` #5 (enum payloads, generics, traits; the part
  on payloads was closed by decision 0017); the master plan's row of
  TASK-20260926-047 and its §5 item 8 (E-2: the effect check is transitive
  through calls, recursion, callbacks and destructors, with a `fail/` test
  per path that exists in the version) and the master prompt as the plan
  cites it (l.176: a generic function does not hide effects); the proxy
  measurements of benchmark B5 (the compiler repository,
  `benchmarks/results/README.md`, "B5 results (generics)", and
  `b5-generics-2026-10-04-aarch64-apple-darwin.json`).
- Spec: new `semantics.md` §2.3 ([GEN-1]–[GEN-12]) when step 2 lands; it
  amends the table of §2, [TY-1], [TY-3], [STRUCT-1], [ENUM-1], [OWN-1],
  [EFF-3], [EFF-5], [DROP-1], [DROP-5], [ABI-2], [GR-1], §12 and the
  grammar of `syntax.md` (part 9). No text of `semantics.md` or
  `syntax.md` changes before step 2.
- Author requirements it follows: E-1 (no new effect), E-2 (part 6), P-2
  (dynamic dispatch, instantiations and their sizes in the facts, part 8),
  P-3 (the measurements compare equivalent contracts and keep measured,
  estimated and not measured apart), M-1 and M-2 (ownership of values of a
  type parameter, part 4: nothing copied, cloned or boxed in silence), Y-5
  and Y-7 (no claim before measurement; predictable costs).

## Context

The language has no user-defined generics. `Option` and `Result` are known
to the compiler ([TY-3]): a type of the compiler (`Ty::Option`,
`Ty::Result`) matched in about 150 places of its passes, with a layout of
its own (`{ tag: u8, value }` and `{ tag: u8, ok, err }`), tags in a fixed
order (`None` 0, `Some` 1; `Ok` 0, `Err` 1) and four rules of the language
that name them: the copy class of [OWN-1] (they copy when every type
argument copies, choice 7 of decision 0020), `require` and `?` ([CELL-4],
[CELL-5]), the expected type of `None`, `Ok` and `Err` ([TY-5]), and text
as their payload ([TEXT-10]). Decision
0022 placed `List<T>` and `Box<T>` in `alloc` and left them waiting for
generics; decision 0016 left the iterator protocol of `for` waiting for
them (OPEN #49); decisions 0017 and 0012 left equality waiting for traits
(OPEN #32).

There are no function values either: every call names its callee, so the
call graph is complete, the effect check of [EFF-3] reads the callee's
signature, and the facts compute `reaches_unsafe` and `allocates` over that
graph ([EFF-5]). E-2 requires the check to stay transitive when callbacks
appear, and the master prompt requires that a generic function never hide
effects.

The task asks to choose between static and dynamic dispatch from
measurements of proxies. Yxy cannot be measured (it has no generics), so
B5 measured the two strategies in two compilers that have both, Rust
(generics against `dyn` trait objects) and C++ (templates against virtual
calls), with the same checks in every column (overflow and bounds checks,
no unwinding, no type information in vtables), on one generic function
instantiated for 1, 16 and 256 distinct types. Measured on one host
(internal, no public claim):

| What | Rust mono | Rust `dyn` | C++ template | C++ virtual |
|---|---|---|---|---|
| a call, instructions / cycles (256 types) | 20.51 / 2.61 | 26.52 / 4.17 | 17.71 / 2.70 | 27.34 / 3.74 |
| compiling to an object, billions of instructions (1 / 16 / 256 types) | 0.85 / 1.56 / 17.37 | 0.85 / 0.96 / 2.60 | 0.45 / 0.94 / 6.35 | 0.45 / 0.54 / 1.91 |
| code (`__text`) added per type, bytes | 141 | 70 | 243 | 89 |
| compile instructions added per type | 64.8 M | 6.8 M | 23.1 M | 5.7 M |

The last two rows are arithmetic on the measured values from 1 to 256
types. A dynamic call cost 27–54% more instructions and 32–83% more cycles
per call at every scale, although its target did not change during a pass
(the best case for the branch predictor), because the call cannot be inlined and the
loop around it cannot be specialized; static dispatch cost compile time and
code in proportion to the number of instantiations, and nothing extra at
one instantiation. Not measured: Yxy, heterogeneous dispatch (a target that
changes from call to call), larger generic bodies, deeper chains of
instantiations, other hosts (`benchmarks/results/README.md`, "Not measured
(B5)"). Estimated: nothing in numbers for Yxy; its compiler hands clang
textual IR at `-O2`, so the C++ column is the closer proxy for the part of
an instantiation's cost that clang pays, and nothing measures the part its
own front end would pay.

## Decision

### 1. Generic items ([GEN-1], [GEN-2])

1. **[GEN-1]** A function, a struct or an enum may declare **type
   parameters**, written after its name: `fn swap<T>(inout a: T, inout b:
   T) effects {}`, `struct Pair<A, B> { first: A, second: B }`, `enum
   Either<A, B> { Left(A), Right(B) }`. A type parameter is a name of a type
   inside its item, and repeats no name in scope (E0202, as a local does).
   Every type parameter of a function appears in the type of a parameter or
   in the result type, and every type parameter of a struct or an enum in a
   field or the data of a variant; otherwise it is an error (E0396), since
   nothing could determine it. Rules on structs and enums hold for their
   generic forms: a generic struct that contains itself, with any type
   arguments, is recursive ([STRUCT-2]).
2. **[GEN-2]** **Type arguments** are written in types (`Pair<u8, bool>`,
   `Either<Point, u32>`), with exactly as many as the item declares
   (E0203). In expressions they are never written: the type arguments of a
   call, a struct literal, a variant or a function used as a value are
   determined by unifying the declared types with the types of the
   arguments, from left to right, and then the result with the expected
   type, as `None` takes its type ([TY-5]). A literal has no type of its
   own ([TY-4]): `id(1)` is determined only by an expected type. A type
   parameter that neither determines is an error (E0391), fixed by writing
   the expected type (`x: u8 := id(1)`). Since `a < b > c` is already an
   error (`syntax.md` §4: comparisons do not chain), written type
   arguments in expressions (`id<u8>(1)`) can be added later without
   changing a valid program (OPEN #5).

### 2. Admissible type arguments ([GEN-3])

**[GEN-3]** A type argument is a type that may be the type of a struct
field: `bool`, the integers, the floats, `()`, an enum, a struct, an
`Option` or a `Result` of those, an instance of a generic struct or enum,
and a function type (part 5). An array, a view (`&[T]`, `&mut [T]`), text
(`&str`) or a capability (`Console`, `Mmio`) is not (E0390), written or
determined. So a value of a type parameter can always be stored, moved and
returned, and no capability travels hidden in one ([CON-1], [MMIO-2]):
the parameters of a function still show every capability it can reach.
`Option` and `Result` keep their own rule ([TEXT-10]: text where text may
go), part 7.

### 3. One check, then static dispatch ([GEN-4], [GEN-6])

1. **[GEN-4]** A generic item is checked **once, in its generic form**. In
   its body a type parameter is an opaque type: a value of it may be
   declared, moved, borrowed, given to a parameter (borrowed, `take` or
   `inout`), returned, stored in a field of a generic struct, in a variant,
   an `Option`, a `Result` or an array literal, matched through those
   (`Some(x)`), and viewed as an element of `&[T]`. No other operation
   applies to it (E0392): no operator, `==`, `.copy()`, `[v; N]`, field,
   `.len`, `match` on it, call, conversion. There are no constraints in this
   version (part 10). An instantiation adds no error to those of the
   generic form, except the limits of [GEN-9]: a generic function that
   checks is valid for every admissible type argument.
2. **[GEN-6]** **Static dispatch, always**: every use of a generic function
   or type names an **instantiation**, the item with its type arguments
   substituted, compiled as code and laid out as a type of its own
   (monomorphization; layouts by [STRUCT-8] and [ENUM-5], per target). An
   instantiation is identified by the ID of the item and the IDs of its
   type arguments, and the program has one of each, whatever number of
   packages asks for it. An instantiation has no hidden dictionary, box,
   allocation or indirect call: a call in it is static exactly when it is
   static in the generic body. Dynamic dispatch exists only where the
   program writes it (part 5).

### 4. Ownership of values of a type parameter ([GEN-5])

**[GEN-5]** For the analysis of moves and destruction (decisions 0020 and
0021) a type parameter is a **move type that may have a destructor**, the
most demanding class: every rule of [OWN-2]–[OWN-10], [DROP-3], [DROP-4]
and [DROP-6] holds in a generic body for every instantiation. A value of a
type parameter moves where a struct moves; a use after a move, a
maybe-moved value and a maybe-moved value where it would be destroyed are
errors (E0360, E0361, E0377), also when every instantiation of the program
is a copy type; nothing copies it (`.copy()` is E0392, and so is `[v; N]`);
`take` and `inout` apply to it (E0365 does not). An instantiation with a
copy type keeps the meaning: moving a value of a copy type copies it, and
destroying it does nothing. An instantiation with a type that has a
destructor destroys its values where the generic body's analysis placed the
destructions ([DROP-3]), with no flag at run time and no copy: nothing is
inserted that the generic form did not show (M-2). A generic struct and a
generic enum with data are move types ([OWN-1], K0), whatever their type
arguments; `Option` and `Result` keep their rule (part 7). A destructor of
a generic struct has exactly the struct's type parameters: `drop fn
free<T>(v: Stack<T>) effects {} { … }` ([DROP-1]).

### 5. Function types and values: the only dynamic dispatch ([GEN-7])

**[GEN-7]**

1. A **function type** is written `fn(P1, P2) -> R effects { e1 }`, each
   parameter a type with its mode (`take T`, `inout T`, or borrowed), the
   result optional (`()`), and the effects clause mandatory (E0111), as in
   a declaration. Two function types are the same type when their modes,
   types, results and sets of effects are equal; there is no subtyping
   between function types.
2. A **function value** is a Yxy function of the program named where a
   value is expected (`f`, `pkg.f`), with no call: `cmp := less`,
   `sort(inout xs, less)`. Its type is the function's signature, or, where
   an expected function type is given, that type ([GEN-8] item 1). A generic
   function becomes a value of one instantiation, which the expected type
   determines (E0391 without one). An `extern` or `export` function, a
   destructor, a variant, an operation of the language (§6.4–§6.6), an
   operation of a capability and `.copy` are not function values (E0397).
   There are no closures: a function value captures nothing.
3. A function value is a **copy type** ([OWN-1]), one code address, which
   may be a parameter, a local, a field, an element, a variant's data or in
   an `Option` or a `Result`. It has no `==` (E0304) and no `match`
   (E0501): comparing code addresses is not a meaning of the program. It
   does not cross the C boundary (E0311, [ABI-2]).
4. **A dynamic call** is a call of a local or a parameter of function type,
   `f(x)` ([GR-1] widened; a field is called through a local, `g :=
   s.on_close` then `g(x)`, so that every callee stays a name). It is the
   only dynamic dispatch of the language: what it runs is not known where
   it is written, the call is not inlined unless the optimizer proves its
   target, and the facts report it (P-2, part 8). An "interface" is a
   struct of function values, written and reported as such.

### 6. Effects through generics and function values (E-2, [GEN-8])

**[GEN-8]** The effects of every call are known from the signature of
what is called, its type arguments and, for a dynamic call, the function
type; nothing in a generic body or behind a function value performs an
effect its caller does not declare:

1. **Function value** (E0614). A function becomes a value of a function
   type only if the type lists every effect the function declares: the
   effects of a function type are an upper bound of every function that a
   value of it can hold. Its own effects may be fewer (`print_all(inout
   xs, double)`, `double` with `effects {}`, into a `fn(u32) -> u32 effects
   { console }`): the function is converted to the type; values of two
   different function types never are.
2. **Callback, and every dynamic call** (E0600, widened within its class).
   A call of a value of function type performs exactly the effects of its
   type, which the caller declares, as [EFF-3] for a static call. A
   function that takes a callback and calls it declares the callback's
   effects; a function that only passes it on does not.
3. **Generic call** (E0600, widened within its class). A call of a generic
   function performs its declared effects and, for each type argument `A`,
   the **destruction effects** of `A`: the effects of the destructors that
   destroying a value of `A` may run ([DROP-5]: its own destructor, those
   of the types it holds). The caller declares them, whether or not the
   body destroys a value of that type parameter: the signature and the type
   arguments tell the effects, never the body. Inside a generic body, the
   destruction effects that come from the body's own type parameters (a
   call `f<T>(…)`, or `f<Pair<T, u8>>(…)`, in `g<T>`) are accounted for by
   the callers of `g`'s instantiations, transitively; those of the concrete
   types in a type argument (`f<Pair<T, Logger>>(…)`: `Logger`'s) the body
   declares, as any function declares the destruction of a `Logger`. The
   same holds for the destructor of a generic struct: destroying a
   `Stack<A>` performs the destructor's effects and those of `A` ([DROP-5]
   as it stands).
4. **Recursion** holds through declarations, also through function values
   (a function that passes itself as a value): every check uses declared
   effects and the effects of function types, never an inferred set.

The compiler reports the missing effect at the call, naming its source: the
function type of the callee, or the type argument and the destructor whose
effect it is.

### 7. `Option` and `Result` ([GEN-12])

**[GEN-12]** `Option` and `Result` stay known to the compiler in step 2,
and move to `core` afterwards, as `pub enum Option<T> { None, Some(T) }`
and `pub enum Result<T, E> { Ok(T), Err(E) }`, in the prelude with their
variants ([PRG-2]), when that move can be made with no change to the code
or the facts of any program (OPEN #52). Their layout is already that of
an enum with data of the same shape (a tag, then every payload, [ENUM-5]),
and their tags are the order of those declarations. Four properties are the
language's and stay on them by name, as lang items: their copy class
([OWN-1], choice 7 of decision 0020: they copy when every type argument
copies; every other generic enum moves), `?` and `require` ([CELL-4],
[CELL-5]), the expected type of `None`, `Ok` and `Err` ([TY-5], now the
rule of [GEN-2] for every generic enum), and text as a payload
([TEXT-10], the one exception to [GEN-3]). Step 2 proves the move is
meaning-preserving for layout: a user `enum Opt<T> { None, Some(T) }` and
`enum Res<T, E> { Ok(T), Err(E) }` have, for every type argument of
`Option` and `Result` in the compiler's corpus, the sizes, alignments,
offsets and tags of `Option` and `Result` on every target.

### 8. Limits and facts ([GEN-9]–[GEN-11])

1. **[GEN-9]** **Monomorphization is bounded**, by counts that depend only
   on the program, never on the machine or on time ([TGT-2]):
   - the **depth** of a chain of instantiations, each requested by the one
     before from a function that is not generic (`main`, an `export fn`, a
     plain function), is at most **64**; one more is an error at the call
     that requests it, with the chain (E0394). Polymorphic recursion
     (`f<T>` calling `f<Option<T>>`) is refused by this bound;
   - the **number** of distinct instantiations of functions and types in a
     program is at most **16 384** (E0395).
   The compiler visits instantiations in a fixed order (the order of
   packages, items and calls in the source), so the same program gets the
   same error at the same site everywhere. An instantiated type also keeps
   the limits of written types ([GR-5], [STRUCT-9]). The bounds refuse
   runaway expansion, not measured programs: B5's proxies took 23–65 million
   compiler instructions per instantiation of a small function, so 16 384
   of them would take minutes in those compilers.
2. **[GEN-10]** **Nothing generic at a boundary**: an `extern fn`, an
   `export fn` and `main` have no type parameters (E0393), and a function
   type is not a type of the C boundary (E0311).
3. **[GEN-11]** **Facts** (`yxy inspect --json`, P-2):
   - each function and type says its type parameters; a generic function
     has no `frame_bytes` of its own;
   - `instantiations[]` lists every instantiation of a function, with its
     type arguments (as types by ID), its effects (declared, and the
     destruction effects of its type arguments, each with the destructor it
     comes from), its `frame_bytes`, the number of its MIR statements (a
     measure of its code before optimization, not of machine code), its
     depth ([GEN-9]) and the call sites that request it;
   - instantiated structs and enums are listed with their layout (size,
     alignment, offsets) per target, as written ones are;
   - each entry of `calls[]` says its dispatch, `static` (with the
     instantiation it names, for a generic callee) or `dynamic` (with the
     function type and its effects);
   - each function lists the functions it takes as values, with the site
     and the function type;
   - `reaches_unsafe` and `allocates` stay upper bounds: a dynamic call may
     reach every function of the program taken as a value of its function
     type, and `limits[]` gains an entry `dynamic_calls` that says so (the
     call graph is no longer exact once a program takes a function as a
     value).

### 9. Grammar, and the rules it amends

The grammar of `syntax.md` §3 gains:

```ebnf
fn_decl     = [ "export" | "extern" | "drop" ] "fn" IDENT [ type_params ]
              "(" [ params ] ")" [ "->" type ] effects [ body ] NL ;
struct_decl = "struct" IDENT [ type_params ] "{" … ;   (* the rest as now *)
enum_decl   = "enum" IDENT [ type_params ] "{" … ;
type_params = "<" IDENT { "," IDENT } [ "," ] ">" ;
type        = … | "fn" "(" [ fn_param { "," fn_param } [ "," ] ] ")"
              [ "->" type ] effects ;                   (* [GEN-7] *)
fn_param    = [ "take" | "inout" ] type ;
```

A function's name where a value is expected is a `primary` already
(`IDENT`, or an import's name, `.` and the name). A function that returns a
function value writes the effects of the result type first: `fn pick(b:
bool) -> fn(u32) -> u32 effects {} effects {}`. `<` after a name in an
expression stays the comparison ([GEN-2]).

When step 2 lands, the text of `semantics.md` changes with it: a new §2.3
holds [GEN-1]–[GEN-12]; the table of §2 gains type parameters and function
types; [TY-1] (function types are value types), [TY-3] (user generics
exist; `Option` and `Result` stay known to the compiler), [STRUCT-1] and
[ENUM-1] (fields and data of a type parameter or a function type), [OWN-1]
(function types are copy types), [EFF-3] (dynamic calls and generic calls,
[GEN-8]), [EFF-5] (the call graph is an upper bound once a function is a
value), [DROP-1] and [DROP-5] (destructors of generic structs), [ABI-2]
(function types do not cross), [GR-1] (a local or parameter of function
type is called, and a function is named as a value) and §12 (user-defined
generics and function values leave the list; traits, closures and method
calls stay).

### 10. Deferred (OPEN)

Not in this decision, each recorded in `OPEN.md`: constraints on type
parameters (traits, interfaces, or the compiler's own `copy`, `eq`, `ord`;
equality of enums with data, #32; the explicit copy of a value of a type
parameter), so the operations of [GEN-4] beyond moving and borrowing; type
arguments written in expressions; effect parameters (a generic function
whose effects are those of the callback it receives); closures; trait
objects or interface values beyond structs of function values; function
values at the C boundary (C callbacks); subtyping between function types;
type parameters for arrays, views, text and capabilities; constant
parameters (`[T; N]` with `N` a parameter); specializing a generic function
on a function argument known at compile time; the move of `Option` and
`Result` (#52); `List<T>` and `Box<T>` of decision 0022 (which need generic
code in `alloc`); the iterator protocol of `for` (#49).

### 11. What step 2 implements, and its gate

Step 2 of TASK-20260926-047 implements parts 1–9 and nothing of part 10.

| # | Work | Rules |
|---|---|---|
| 1 | Lexer, parser and formatter: type parameter lists on `fn`, `struct`, `enum` and `drop fn`; function types; functions as values; the canonical layout and its idempotence | [GEN-1], [GEN-7] |
| 2 | Names and types: type parameters in scope, unification of type arguments with arguments and the expected type, admissible type arguments, the operations of a type parameter, function types and values, dynamic calls, the boundary | [GEN-1]–[GEN-4], [GEN-7], [GEN-10] |
| 3 | Ownership on the generic form: the analyses of moves, destruction and views with a type parameter as a move type that may have a destructor | [GEN-5] |
| 4 | Effects: function values against their types, dynamic calls, the destruction effects of type arguments at generic calls and at destructions of generic structs | [GEN-8] |
| 5 | Monomorphization in the MIR, code generation (an indirect call per dynamic call, none elsewhere) and the reference evaluator, with deterministic symbol names and the bounds | [GEN-6], [GEN-9] |
| 6 | Facts and their documentation (`docs/json.md`), the new and widened diagnostic codes (`docs/diagnostics.md`), `STATUS.md`, and the text of `semantics.md` and `syntax.md` | [GEN-11], part 9 |
| 7 | The layout equivalence of `Option`/`Result` and user enums of the same shape | [GEN-12] |

The gate, with at least one positive (`run/` or `pass/`, the three engines
agreeing: native `-O0`, `-O2` and the evaluator) and one negative (`fail/`
with its exact diagnostics) test per rule:

- **[GEN-1]**: generic functions, structs and enums in `run/`; an unused
  type parameter (E0396); a type parameter that repeats a name (E0202); a
  generic struct that contains itself through a type argument (the error of
  [STRUCT-2]).
- **[GEN-2]**: type arguments determined by arguments, by the expected
  type, and by both; `id(1)` with no expected type, `None`-like variants of
  a generic enum and a generic function as a value without one (E0391); a
  wrong number of type arguments, and arguments to a type that has none
  (E0203).
- **[GEN-3]**: each refused kind as a written and as a determined type
  argument (array, `&[u8]`, `&mut [u8]`, `&str`, `Console`, `Mmio`: E0390);
  a function type as a type argument in `run/`.
- **[GEN-4]**: each refused operation on a value of a type parameter
  (E0392); every allowed one in `run/`; a generic body with an error
  reported once, with no instantiation in the program.
- **[GEN-5]**: use after move, maybe-moved and maybe-moved at a destruction
  of a value of a type parameter (E0360, E0361, E0377) in a program whose
  only instantiations are copy types; destruction order of values of a
  type parameter with a destructor on every exit (`require`, `?`, `return`
  in an arm, `break`, the end of a body), counted by the test hooks, in the
  three engines; `take` and `inout` of a type parameter.
- **[GEN-6]**: one generic function at many types, across packages (a
  `tests/modules` case), with one instantiation per distinct type
  arguments in the facts; the IR of a program without function values has
  no indirect call; a program without generic items compiles to the same
  IR and facts as before step 2.
- **[GEN-7]**: function values in a local, a parameter, a field, an array,
  an `Option`; one indirect call in the IR per dynamic call site; an
  `extern`, `export` or `drop` function, a variant and an operation of the
  language as values (E0397); a field called directly (`s.f(x)`, the
  method call of E0900); a function type in an `extern` signature (E0311);
  `==` on function values (E0304, operator not defined for type, widened
  within its class).
- **[GEN-8]**, E-2, one `fail/` per path, each with its positive twin that
  declares the effect:
  - **callback**: a function that calls its parameter of type `fn(u32)
    effects { console }` and declares `effects {}` (E0600 at the call);
  - **function value**: a function that declares `console` given as a value
    of `fn() effects {}` (E0614), and a value of `fn() effects { console }`
    kept in a struct and called, through a local, in a function that
    declares `effects {}` (E0600);
  - **generic call**: a generic function called with a type argument whose
    destructor performs `console`, from a function that declares `effects
    {}` (E0600 at the call, naming the destructor), also when the generic
    body never destroys a value of that type; a generic function that calls
    a callback parameter without declaring its effects (E0600); a
    `Stack<Logger>` destroyed in a function that declares `effects {}`
    (E0600, through the generic destructor);
  - **recursion**: a function that passes itself as a value and is called
    through it, without declaring the effect of its type (E0600).
- **[GEN-9]**: polymorphic recursion (E0394, with the chain, the same on
  every target); a generated program with 16 385 instantiations (E0395) and
  one with 16 384 (valid); a chain of depth 64 (valid) and 65 (E0394).
- **[GEN-10]**: a generic `extern fn`, `export fn` and `main` (E0393).
- **[GEN-11]**: the facts of a fixture derived by hand: instantiations with
  their type arguments, effects, `frame_bytes`, depth and requesting sites;
  `dispatch` of each call; the functions taken as values; `reaches_unsafe`
  true for a function whose only way to an unsafe region is a dynamic call,
  through a function of the program taken as a value of that call's
  function type; `allocates` likewise; the `limits` entry.
- **[GEN-12]**: the layout equivalence above, on the five code-generating
  targets.

B5 gains a Yxy column when step 2 lands (static and dynamic, the same
contract), and the proxies stay as they are.

### 12. Diagnostic codes

New codes, in free ranges of the compiler's `docs/diagnostics.md` (types,
E039x; effects and capabilities, E061x):

| Code | Title | Rule |
|---|---|---|
| E0390 | type argument not allowed | [GEN-3] |
| E0391 | type argument not determined | [GEN-2], [GEN-7] |
| E0392 | operation not available on a type parameter | [GEN-4], [GEN-5] |
| E0393 | generic item at a boundary | [GEN-10] |
| E0394 | instantiation too deep | [GEN-9] |
| E0395 | too many instantiations | [GEN-9] |
| E0396 | unused type parameter | [GEN-1] |
| E0397 | not a function value | [GEN-7] |
| E0614 | function value with effects beyond its type | [GEN-8] |

By the rules of the codes (`docs/diagnostics.md`, "Codes and their
meaning"): E0600 (undeclared effect) widens within its class (a dynamic
call; the destruction effects of a type argument at a generic call); E0203
(unknown type) widens within its class (type arguments of a generic type
and of a type parameter); E0202 (name already declared) to type parameters;
E0311 (type not allowed at the C boundary) to function types; E0304
(operator not defined for type) to `==` and the other operators on function
values; E0501 (pattern does not match type) to a `match` on a function
value; E0111
(missing effects clause) to function types; E0900 narrows (generic
functions, structs and enums); E0206 (not callable) narrows (a function
named without a call is a function value where a value is expected; calling
a variable that is not of function type keeps E0206).

## Alternatives

- **Dynamic dispatch by default** (one compiled body for every type
  argument, values passed boxed or with a dictionary of operations, as
  Java, OCaml or unspecialized Swift): one copy of code and no growth with
  instantiations (B5: 70 and 89 bytes per type against 141 and 243), but a
  call that is not inlined at every use of an operation of a type parameter
  (B5: 27–54% more instructions, 32–83% more cycles per call, in the best
  case), and boxing or dictionaries that the program did not write (M-2,
  P-2). Rejected as a default.
- **Dictionaries per layout** (Go's stenciling by GC shape: one copy per
  layout, with a dictionary for the operations): fewer copies than full
  monomorphization, but the indirect calls through the dictionary are
  hidden in the generated code; a later implementation strategy could share
  instantiations of equal code without changing the language, if it
  reports them. Not adopted.
- **Checking each instantiation instead of the generic form** (C++
  templates before concepts, Zig's `comptime`): accepts more programs (any
  operation a type argument happens to support), but errors appear in the
  body when a caller instantiates it, so a generic function's signature
  does not say what it accepts, and a change elsewhere breaks it. Rejected
  ([GEN-4]); constraints will say what a body may do with a type parameter.
- **Traits or interfaces now** (Rust's traits, Go's interfaces, Swift's
  protocols): the operations of a type parameter, `==` and ordering, and
  dynamic dispatch through interface values. A larger design (coherence,
  where implementations live across packages, defaults, the copy class as a
  trait, K1 of decision 0020); kept for later (OPEN #5) so that this step
  stays implementable in one step. Until then an operation a generic
  function needs is passed as a function value (a comparison to a sort),
  whose dynamic call the facts report.
- **Built-in constraints only** (`copy`, `eq`, `ord` known to the
  compiler): small, but a second mechanism beside traits once those come;
  deferred with them.
- **Trait objects (`dyn Trait`)** as the written form of dynamic dispatch:
  needs traits; a struct of function values is the explicit form until
  then, and every call through one is a reported dynamic call.
- **A marker at the dynamic call** (`call f(x)`, `f.call(x)`): it would show
  dynamic dispatch at the call as `inout` shows the borrow; the callee of
  a dynamic call is always a local or a parameter whose function type is
  written in the signature or the declaration, and the facts list each one.
  Adding a marker later breaks every dynamic call, so it is a choice for
  the author's review.
- **Effects of type arguments' destruction as a bound** (a type parameter
  admits only types whose destruction has no effect, unless it says so):
  refuses `List<File>` where `File`'s destructor calls C (`ffi`), and any
  generic container of such values. **Effects inferred from the body**
  (only the types a body destroys count): fewer declarations, but the
  effects of a call would depend on the callee's body, against decision
  0005 (signatures state their effects). The caller pays instead ([GEN-8]
  item 3): conservative, decided by the signature and the type arguments.
- **Effect parameters** (`fn map<T, U, effects E>(…, f: fn(T) -> U effects
  { E }) effects { E }`, as in Koka, or Swift's `rethrows`): one `map` for
  every callback; without them, a library function that calls a callback
  fixes the callback's effects. A real cost for a library, deferred because
  it changes the effect check of every call (OPEN #53).
- **Subtyping between function types** (a `fn() effects {}` value where a
  `fn() effects { console }` is expected): variance in the type system for
  one convenience; a named function converts already (item 1 of [GEN-8]).
- **Closures** (function values that capture variables): ownership of the
  captured values (moved, borrowed, for how long) is a design of its own
  under decision 0020; deferred (OPEN #53).
- **Moving `Option` and `Result` to `core` in step 2**: one representation
  less in the compiler, but a change of about 150 places of it in the same
  step as generics; the layout test of [GEN-12] makes the later move a
  refactoring whose gate is unchanged code and facts.
- **No bound on instantiations, or a bound in time**: polymorphic recursion
  would not end, and a bound in time is not deterministic ([TGT-2]); the
  bounds of [GEN-9] are counts, like [MATCH-6]'s budget. Rust's default
  recursion limit is 128; 64 levels of a chain started from a plain function
  is ample for the corpus (which has none) and leaves room for the type
  nesting of [GR-5].

## What could change it

- The author's review: static dispatch as the only dispatch of generic
  code, a dynamic call written without a marker, the caller paying the
  destruction effects of type arguments, the bounds of [GEN-9].
- Step 2's Yxy column of B5: if Yxy's cost per instantiation is far from
  the proxies', the bounds and the default may be revisited.
- Programs that need constraints, effect parameters or closures often
  enough (OPEN #5, #53); the library's `List<T>`, `Box<T>` and the iterator
  protocol (#49), which are the first consumers.
- Heterogeneous dispatch measured (a target that changes at every call),
  where dynamic dispatch costs more than B5 measured.
