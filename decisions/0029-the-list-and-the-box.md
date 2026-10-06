# Decision 0029: The list and the box

- Status: Experimental — not accepted. Every choice below (the operations,
  the view of the elements, the type of the error of a `try_` operation that
  takes a value, the growth, the order of destructions, the amendments of
  [REF-2] and [GEN-1]) is the orchestrator's, with the alternatives that
  follow, for the author's review; its state is controlled in the register of
  the control repository (`plans/decisions/README.md`).
- Date: 2026-10-06
- Origin: decision 0022, part 5 (the place of `alloc/list` and
  `alloc/boxed` and the shape of their plain and `try_` operations, with the
  type of the error of a `try_` operation that takes a value left to be
  decided with them); decision 0025, part 10 (`List<T>` and `Box<T>`
  deferred until the library has generic code); `OPEN.md` #5 and #52; the
  basic library promised for 0.2 (the compiler's `STATUS.md`, R1182).
- Spec: new `semantics.md` §4.5 ([LIST-1]–[LIST-6], [BOX-1]–[BOX-3]);
  amends [GEN-1], [REF-2], [TEXT-8], [TEXT-9], [ALLOC-3], the introduction
  of §7.2 and §12; `modules.md` [STD-8]; `OPEN.md` #5 and #52; decisions
  0022 (part 5) and 0025 (part 10) gain a line that points here.
- Implementation: the compiler's `docs/implementation/STATUS.md`,
  R1500–R1519 (R1182 executed).
- Author requirements it follows: M-1 and M-2 (nothing copied, cloned or
  boxed in silence: a value goes onto the heap only through a call written
  in the program), M-4 (no abort hidden behind an API presented as
  recoverable), P-2 (every allocation in the facts), E-2 (the destruction
  effects of the values a list holds are its user's, through the generic
  destructor), Y-7 (predictable costs: a documented growth).

## Context

Decision 0022 placed a growable list and a single owned value on the heap in
the `alloc` layer, with the policy O2 for running out of memory, and left
them waiting for generics; decision 0023 built the owned string the way
they would be built — a struct of the library with private fields and a
destructor, over a few operations of the library's own — and decision 0025
gave the language generic structs and functions, checked once in their
generic form and instantiated by monomorphization, with **no constraints**:
a generic body may only move, borrow, store and destroy a value of a type
parameter ([GEN-4], [GEN-5]).

Four things had to be decided to write them as library code under those
rules:

1. **Where the values are.** A struct's fields hold its values inline; a
   list's values are on the heap, in a block whose size changes. The
   library needs operations that move a value of a type `T` into a slot of
   a block and out of it, and the size of a value of `T`; and a struct whose
   type parameter no field holds (`List<T>` holds an address, a length and
   a capacity), which [GEN-1] refused (E0396).
2. **How an element is read without moving it.** The language has no
   reference to one value: its borrows are a parameter without `take`
   ([OWN-3]), a binding of a `match` ([OWN-5]), `inout` ([OWN-9]) and the
   views of arrays, `&[T]` and `&mut [T]`, which no function returned
   ([REF-2]). A `get(l, i) -> Option<T>` would copy the element, which
   needs a constraint (OPEN #5); a callback `with(l, i, f)` would fix the
   callback's effects (no effect parameters, OPEN #53) and could not carry
   state (no closures).
3. **The error of a `try_` operation that takes a value** (decision 0022,
   part 5): when `try_push(inout l, take v)` cannot allocate, `v` must come
   back, or the failure would destroy it.
4. **When the values are destroyed**, with which effects, and in which
   order.

## Decision

### 1. Packages and types ([LIST-1], [BOX-1], [STD-8])

| Package | Type | What it is |
|---|---|---|
| `alloc/list` | `List<T>` | A growable sequence of values of `T` on the heap, in one block |
| `alloc/boxed` | `Box<T>` | One value of `T` on the heap |
| `alloc` | `Refused<T>` | The error of a `try_` operation that takes a value: `pub value: T`, `pub error: AllocError` |

`List<T>` and `Box<T>` are structs of the library with private fields (the
address of the block, and for the list its length and capacity) and a
destructor ([DROP-1]): they move, are destroyed by their owner on every
exit ([DROP-3]) and are never copied (`.copy()` and `[v; N]` of one are
E0375, [DROP-4]: a deep copy would need a copy of each value, an operation
of `T` that a generic function does not have). `T` is any admissible type
argument ([GEN-3]): never text, an array, a view or a capability, so a list
never stores a view and no capability travels hidden in one. The package of
the box is `boxed`, as decision 0022 named it: a package's local name cannot
be the name of a parameter or local of a file that imports it ([IMP-5]),
and `box` is the name a program gives a box.

### 2. The values on the heap ([GEN-1] amended, [LIST-1], [LIST-2])

- The library builds both types on five operations of its own, names in the
  packages of the standard library only, as decision 0023's heap and text
  operations are: the size of a value of `T`, a view of the first `n`
  values of a block, a value moved into a slot of a block (`take`), a value
  moved out of one, and a check of an index that traps with *index out of
  bounds* (T0004). Their `T` is determined as a generic call's type
  arguments are ([GEN-2]): from the argument that holds it, or from the
  expected type. Moving a value in or out copies its bytes, which the facts
  report (`copies`, reason `heap`), as every move of a value kept in memory
  is.
- **[GEN-1] amended**: a struct of the standard library may declare a type
  parameter that no field mentions, whose values it keeps on the heap
  through those operations; its type arguments are then written or expected
  ([GEN-2]), never determined by a field. Every other package keeps E0396.
  Since no field holds a `T`, a struct that holds a `List<Self>`, a
  `Box<Self>` or an `Option<Box<Self>>` is not recursive ([STRUCT-2]): its
  values are on the heap, so its size is finite (a tree, a linked list).
- A list keeps its values in one block, one after another, each at the
  distance of the size of `T` (a multiple of its alignment); its
  **capacity** counts values. A block is at least one byte, also for a type
  of no bytes (`()`), so that every slot has an address.
- **Growth**: when the block is full, the new one holds **twice as many
  values, at least 4 and at least the new length**, or exactly the new
  length when that larger block cannot be had (it does not fit, or the
  allocator refuses it). `with_capacity(n)` makes room for exactly `n`
  values; `reserve(inout l, n)` grows by the same rule as a push. It is part
  of the documented behaviour (`capacity`), as the string's growth is.

### 3. Operations ([LIST-3], [BOX-2])

`alloc/list` (every function generic in `T`, every one `effects {}`):

| Function | Fails | Does |
|---|---|---|
| `new() -> List<T>` | never | An empty list; allocates nothing |
| `with_capacity(n)` / `try_with_capacity(n)` | `T0007` / `Err(AllocError)` | An empty list with room for `n` values |
| `reserve(inout l, n)` / `try_reserve(inout l, n)` | `T0007` / `Err(AllocError)` | Room for `n` more values |
| `push(inout l, take v)` / `try_push(inout l, take v)` | `T0007` / `Err(Refused<T>)` | Appends `v` |
| `insert(inout l, i, take v)` / `try_insert(…)` | `T0004`, `T0007` / `T0004`, `Err(Refused<T>)` | Puts `v` at `i` (`i <= len`), the values from `i` on one place up |
| `pop(inout l) -> Option<T>` | never | The last value, taken out, or `None` |
| `remove(inout l, i) -> T` | `T0004` | The value at `i`, taken out; the values after it one place down |
| `set(inout l, i, take v)` | `T0004` | Destroys the value at `i`, then puts `v` there |
| `replace(inout l, i, take v) -> T` | `T0004` | Puts `v` at `i`, returns the old value |
| `swap(inout l, i, j)` | `T0004` | Exchanges two values |
| `clear(inout l)` | never | Destroys every value; the capacity stays |
| `len(l)`, `capacity(l)` | never | The number of values; of values it holds before it allocates again |
| `view(l) -> &[T]` | never | The values, viewed (part 4) |

`alloc/boxed`:

| Function | Fails | Does |
|---|---|---|
| `new(take v) -> Box<T>` / `try_new(take v)` | `T0007` / `Err(Refused<T>)` | A box that holds `v` |
| `view(b) -> &[T]` | never | Its value, viewed as the one element of a slice |
| `replace(inout b, take v) -> T` | never | Puts `v` in the box, returns the old value |
| `into_inner(take b) -> T` | never | Its value, taken out; the block is freed |

An index that is not below the length (above it for `insert`) traps with
*index out of bounds*, T0004, at the check in `alloc/list`, as `a[i]` of an
array does ([REF-4]): the recoverable form of an index is the program's
comparison with `len(l)` first. Every function that can trap says so, with
its codes ([ALLOC-4], M-4).

### 4. A borrowed view of the elements ([LIST-5], [BOX-2], [REF-2] amended)

A reference to one element does not exist in the language, and none is
added: the sound way to read elements without moving them is the view the
language already has, a slice `&[T]` of all of them. `list.view(l)` and
`boxed.view(b)` return one, which needs one rule amended:

- **[REF-2] amended**: a function may return a slice `&[T]` (never a view
  `&mut [T]`, which stays a parameter or a local). It follows the rule of
  text ([TEXT-8], [TEXT-9]): the result of a call views what its arguments
  view, but those of `take` parameters, and a function returns only a slice
  that views no local of its own, `take` parameter or `inout` parameter
  (E0750). So `view(l)`, given the list `l` that a function owns, views
  `l`; `fn items(l: list.List<u8>) -> &[u8] { return list.view(l) }` is
  valid, its caller's result viewing the caller's list.
- While the view lives `l` is **frozen** ([TEXT-8], [REF-7]): pushing,
  setting, clearing or moving it, which could move or free the block under
  the view, is an error (E0367, E0363). A view held by a variable lives
  until the end of its block; given to a call, until the call returns.
- An element `xs[i]` of the view is read as an element of any slice: a
  value of a copy type is copied, one of a type that moves is borrowed — a
  field read, the argument of a parameter without `take`, the source of
  another view (`list.view(xs[i])` of a list of lists, `string.as_str(xs[i])`
  of a list of strings) — and never moved out (E0362). A value is changed or
  taken out through the list itself, which the function holds `inout`:
  `set`, `replace`, `swap`, `remove`, `pop`.

### 5. Out of memory ([LIST-4], [BOX-3], [ALLOC-3] amended)

As decision 0022 says: the plain form traps with *allocation failed*
(T0007) at the operation in the library when the allocator has no memory or
the size does not fit (above the largest `isize` of the target, or a count
of values whose bytes overflow `usize`); the `try_` form never traps for
lack of memory and leaves its `inout` list as it was. Its error is
`alloc.AllocError` when it takes no value (`try_with_capacity`,
`try_reserve`) and **`alloc.Refused<T>`** when it takes one (`try_push`,
`try_insert`, `boxed.try_new`): the value, given back, and the
`AllocError`, in public fields, so that `Err(r)` then `r.value` takes the
value back and `r.error` says why. [ALLOC-3] says so, and the compiler's
check of M-4 accepts either error.

### 6. Destruction ([LIST-1], [LIST-6], [BOX-1], [DROP-5])

- Destroying a list destroys its values **from the first to the last**, as
  an array's ([DROP-2]), then frees its block; destroying a box destroys its
  value, then frees its block.
- `set` destroys the old value **before** it stores the new one, as an
  assignment does ([DROP-3]); `clear` destroys every value at once, the
  first first. A value taken out (`pop`, `remove`, `replace`,
  `into_inner`) belongs to the caller, which destroys it where it destroys
  its values; a value given to `push`, `insert`, `set`, `replace` or `new`
  belongs to the list or the box.
- Effects: the destructor of `List<A>` and of `Box<A>` is the instantiation
  of a generic destructor, so destroying one performs the destruction
  effects of `A` ([DROP-5], [GEN-8] item 3), and every call of a function of
  `alloc/list` or `alloc/boxed` with the type argument `A` performs them
  too: a function that uses a `List<File>` whose destructor performs `ffi`
  declares `ffi`. Nothing of it is decided by the body of the library.

### 7. Facts (P-2)

Nothing new is reported: the allocations are the library's own
(`allocations` of its functions; `allocates` of every function and call
that reaches them), the instantiations of the generic functions and structs
are in `instantiations[]` and the instantiated types with their layouts,
and the moves into and out of the heap of a value kept in memory are
`copies` with the reason `heap`.

### 8. Not in this decision (OPEN #5, #49)

A copy of a list or a box, equality, search, sorting and any operation that
reads a value as more than bytes to move (constraints, OPEN #5; until then a
function value, as decision 0025 says); `for` over a list itself (the
iterator protocol, #49; iterate `list.view(l)`); a view `&mut [T]` of the
elements returned by a function (an exclusive view that outlives its call);
`truncate`, `extend`, `append`, shrinking the block, a list built from an
array; a box of a value whose size is not known (there are no such types).
`Option` and `Result` stay built into the compiler (#52): the library's
`pop` and `try_` forms use them as any program does, and their move to
`core` changes no code of `alloc/list` or `alloc/boxed`.

## Alternatives

- **An element by a copy** (`get(l, i) -> Option<T>`): needs a copy of `T`,
  a constraint (OPEN #5), and copies in silence (M-2). **A callback**
  (`with(l, i, f: fn(T) effects {…})`): the library would fix the effects of
  the callback (no effect parameters, OPEN #53) and the callback could not
  reach its caller's state (no closures). **A reference type** (`&T` to one
  value, with the sources of [TEXT-8]): a new kind of type in every pass of
  the compiler, for what a slice of one or more values already gives. **No
  borrowed access** (only `take`/`replace`/`swap`): sound, but reading a
  value would take it out and put it back. Chosen: the slice, with [REF-2]
  amended under the rule text already follows.
- **Returning slices only from the library** (a privilege of its packages,
  like their own operations): a public signature a program could not write,
  and a program could not wrap `list.view` in a function of its own. The
  rule of [TEXT-9] is the same for any function, so it is amended for all.
- **The error of a `try_` operation that takes a value**: `Result<(), T>`
  (the value back, the reason lost: M-4 wants the reason); two variants
  `CapacityOverflow(T)` and `OutOfMemory(T)` (a second enum of reasons
  beside `AllocError`); a value that is destroyed when the operation fails
  (the program loses what it could not store). Chosen: `Refused<T>`, the
  value and the `AllocError`, public fields of a struct of `alloc`.
- **An index that does not trap** (`set` and `remove` returning `Option` or
  `Result`): every call would need a `match` for an error the program
  usually rules out with `len(l)`; the array's `a[i]` traps (T0004), and
  the list follows it, says so in every function and reports the trap at
  its site.
- **A typed address or a marker field** instead of a struct parameter that
  no field mentions: a field `[T; 0]` holds a `T` for the analyses of
  recursion and layout (a tree of boxes would be E0316) and changes the
  alignment of the list; a field of function type `fn(T)` costs a word and
  a function value per list. A new kind of type (a raw typed address) would
  be one more type in every pass for what only the library does. Chosen:
  [GEN-1] amended for the library's structs only.
- **Rust's growth** (twice, at least 4 or 8 by the size of `T`, and an error
  when the doubled size does not fit): the fallback to exactly the new
  length keeps a list usable up to the largest block the target holds; the
  floor of 4 is the same for every `T`, so that the capacity after a push
  does not depend on the target.
- **`box` as the package's name** (`import "alloc/box"`, `box.new(v)`): the
  most natural variable name of a box would then be refused in every file
  that imports it ([IMP-5]); decision 0022 had already named it `boxed`.

## What could change it

- The author's review of every choice above, at the gate of 0.2.
- Constraints on type parameters (OPEN #5): `copy` of a list, equality,
  search and sorting; the iterator protocol (#49): `for x in l`.
- A reference to one element, or views `&mut [T]` returned, if programs
  that change elements in place through `replace` and `swap` show that they
  cost more than they say.
- The measures of real programs: the growth factor and its floor.
