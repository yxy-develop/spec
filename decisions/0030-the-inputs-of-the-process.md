# Decision 0030: The inputs of the process as capabilities

- Status: Experimental — not accepted. Written for the completion audit of
  2026-10-06 (item 9, "argumentos, ambiente e stdin como capacidades") and
  `decisions/OPEN.md` #55, for the author's review. It follows decision
  0026 to the letter: root capabilities born only in `main`, an effect and
  a capability for each resource, failures as values, no global singleton,
  the capabilities in the facts. Every choice below is the orchestrator's,
  with the alternatives that follow.
- Date: 2026-10-06
- Origin: `decisions/OPEN.md` #55 ("standard input and error as
  capabilities, the arguments and the environment given to `main` as
  capabilities", the audit's §12b.1.3, row 16); decision 0026, part 11.
- Spec: `semantics.md` [PRG-2], the table of §2 (types), §7 ([EFF-1],
  [EFF-2], the table), §7.1 ([CON-1]), §7.5 ([CAP-1], [CAP-3]), new §7.7
  ([ARGS-1], [ENV-1], [IN-1], [ERR-1]), §10 ([MAIN-1]); `modules.md`
  [STD-5], the list of the packages of `std`; `OPEN.md` #48, #55.
- Author requirements it follows: E-1, E-2, E-3 (console, files, network,
  clock and randomness through explicit APIs, without global singletons:
  the inputs of the process are the same kind of resource), Y-9 (no effect
  of a domain), M-2, M-4.

## Context

Before this decision a Yxy program could not read its command-line
arguments, its environment or its standard input, nor write to standard
error, except through `extern fn` and `ffi`, with ambient authority. The
four are resources of the hosted runtime that a caller may want to exclude
(the environment holds secrets, standard input blocks, standard error is a
second stream) and that a test may want to give a function on its own: by
the four questions of decision 0026 each is a capability (Q4) with an
effect (Q3).

## Decision

### 1. The catalogue grows by four entries

| Entry | Effect | Capability value | Failure type | Contract | Q1–Q4 |
|---|---|---|---|---|---|
| arguments | `args` | `Args`: root, the arguments after the program's name | `process.ArgError` (`Missing`, `NotUtf8`) | read only; text views the bytes the process received, in place, for its whole life; bytes that are not UTF-8 are never text ([ARGS-1]) | no, no, yes, yes |
| environment | `env` | `Env`: root, the environment the process received | `process.EnvError` (`NotSet`, `InvalidName`, `NotUtf8`) | read only; the first entry `NAME=value` of that name; a name that is empty or holds `=` or a zero byte is `InvalidName` ([ENV-1]) | no, no, yes, yes |
| standard input | `input` | `Stdin`: root, descriptor 0 | `input.InputError` (`Closed`, `Other(errno)`) | one read of the system per call, 0 at the end, an interrupted read retried ([IN-1]) | no, no, yes, yes |
| standard error | `console` | `Stderr`: root, descriptor 2, with the operations of a console | none returned: a refused write traps, T0005, as the console's ([CON-4]) | every byte written, in order, unbuffered ([ERR-1]) | no, no, yes, yes |

The effects name kinds of action that the hosted runtime mediates (reading
the arguments, reading the environment, reading input), none a domain
(Y-9). Writing to standard error is the same kind of action as writing to
standard output, so it performs `console`; the capability says which
stream (decision 0015: "the capability says which stream, the effect that a
write may happen").

### 2. Effect ≠ capability: the matrix, four new rows

| Resource (operation) | effect yes, capability yes | effect no, capability yes | effect yes, capability no | effect no, capability no |
|---|---|---|---|---|
| arguments (`process.arg_count`) | runs (`run/process_inputs`) | E0600 (`fail/inputs_capability_without_effect`) | E0615 (`fail/inputs_effect_without_capability`) | E0615 and E0600 (`fail/inputs_matrix_neither`) |
| environment (`process.env_var`) | runs | E0600 | E0615 | E0615 and E0600 |
| standard input (`input.read`) | runs: 0 bytes from `/dev/null` | E0600 | E0615 | E0615 and E0600 |
| standard error (`stderr.print`) | runs | E0600 | E0610 (as `console.print` with no `console`) | E0610 |

### 3. No global singleton

The four are born only in `main` ([CAP-2], [MAIN-1]): no expression creates
one (E0611), none is returned, stored, a type argument or given to C (E0612,
E0390, E0311), and the library has no free function that reaches them
([CAP-5]): `std/process` and `std/input` take the capability as their
first parameter. The hosted runtime reads `argv` and `envp` from the C
`main` and gives them to the Yxy `main` as the values of `Args` and `Env`;
no global of the program holds them.

### 4. The API

- `std/process`: `arg_count(args: Args) -> usize`, `arg(args: Args, i:
  usize) -> Result<&str, ArgError>`, `env_var(env: Env, name: &str) ->
  Result<&str, EnvError>` (`var` is reserved, [LEX-7]).
- `std/input`: `read(stdin: Stdin, buf: &mut [u8]) -> Result<usize,
  InputError>`.
- `Stderr`: the operations of a console ([CON-2]), `stderr.print("…")`,
  `print_u64`, `print_i64`, `print_f32`, `print_f64`.
- The toolchain: `yxy run <input> -- <arg>...` gives the program the
  arguments after `--` (and `yxy dev eval`, the reference evaluator, the
  same).

### 5. Facts

`capabilities` lists the four with their effects (`Stderr`: `console`); a
write to standard error is a `capability_calls` entry of `Stderr`; the
types have the kinds `args`, `env`, `stdin`, `stderr`. The measure of part
10 of decision 0026, taken again: a function holds `Args`, `Env` or `Stdin`
exactly when it declares `args`, `env` or `input`; a function that holds
only `Stderr` declares `console` with no `Console`, the case where the
effect no longer says which capability.

## Alternatives

- **One effect `process` for the arguments and the environment** (and
  `input` for standard input): one name less, the capability still says
  which resource; not taken because the environment holds secrets that a
  caller may want to exclude while it allows the arguments (as Deno's
  `--allow-env`), and decision 0026 gives each resource its own effect.
- **`console` for standard input too**: one effect for the terminal; not
  taken: reading may block for ever and is not the kind of action of
  writing, and a caller that allows output (logging) may exclude input.
- **A new effect for standard error** (`stderr`): taken neither: it is the
  same action as the console's, and the capability already says which
  stream.
- **Standard error as a second `Console`**: every function of `std/print`
  and `std/capture` would take it; but [MAIN-1] gives each root capability
  by its type, so `main` cannot tell two consoles apart without positions
  or names, which it does not use. **A `Console` carrying both streams**:
  then a function given the console could write to either, against "the
  capability says which stream". **A console for standard error derived by
  a callback** (as `capture.text`): a function value captures nothing
  ([GEN-7]), so a program could not pass its state. `Stderr` with the
  operations of a console is taken; the price: `std/print` takes a
  `Console` only, and a test cannot capture standard error (OPEN #55).
- **The arguments and the environment as values given to `main`**
  (`fn main(args: &[&str])`): no capability, no effect, against E-3 (any
  function given the slice reads them, and nothing in the signature of its
  callers says so); and text of the arguments that are not UTF-8 would have
  no place.
- **The program's name as argument 0** (C's `argv[0]`, Go's `os.Args`):
  not taken: it is set by whoever starts the process, differs between
  `yxy run` (a temporary path) and the evaluator, and is rarely what a
  program needs; deferred.
- **Bytes of the arguments and variables**: a function cannot return a
  view of bytes in this version (E0402), and a copy into a buffer would need
  a copy that never traps; text is given, and `NotUtf8` says when there is
  none. Deferred (OPEN #55).
- **`getenv`**: the environment is read from `envp` instead, so that its
  text views memory that lives as long as the process and no buffer bounds
  a name; a change of the environment by foreign code is outside the
  guarantees of [EFF-5].
- **Standard input filled until the buffer is full or the input ends** (as
  `fs.read`): a line typed at a terminal would wait for more; one read of
  the system is the usual contract, and a loop reads it all.

## What could change it

The author's review: the names of the effects (`args`, `env`, `input`) and
of the capabilities (`Args`, `Env`, `Stdin`, `Stderr`, four new prelude
names, OPEN #48), `Stderr` with the operations of a console against a
second `Console`, the program's name, and the bytes of what is not UTF-8.
