# Yxy packages, modules and imports — experimental draft

Status: **experimental; partly implemented.** This is the normative draft of
the package and module system. Packages of one module (§1–§5, §9) are
implemented and integrated into `syntax.md` and `semantics.md`; dependencies
from local directories and Git origins, the lock, minimal version selection,
`yxy lock`, `yxy fetch`, `--locked` and `--offline` (§6, §7) are implemented
in part; `add`, `update`, `remove`, the package service and §8 are not. See
"Implementation status" in §10.

Statements marked **(author)** are the author's direction, recorded in
`decisions/0011-imports-by-origin.md`. **Every other rule in this document is
a proposal of this draft**: experimental, open to the author's review, and
changed only through a recorded decision. Rules that restate the author's
bootstrap instructions (§7, §14, §15: no module initializers with arbitrary
effects, no build scripts, no silent downloads, no dependencies guessed by
an assistant, the `core`/`alloc`/`std` layering) cite them; those
instructions present such points as guidelines to validate, so they are
experimental here too. Open questions are in `decisions/OPEN.md` (#15–#31).

Rule identifiers such as `[IMP-3]` are stable anchors. They are unrelated to
the finding identifiers of the review that motivated this draft (`PKG-01` to
`PKG-03`, `CACHE-02`, in the control repository,
`plans/reviews/2026-09-25-cache-and-packages-review.md`).

Module paths in examples (`github.com/example/…`) are fictitious.

## 0. Author direction

- **[AUTHOR-1] (author)** Packages are imported by a path associated with
  their origin, the repository they come from, inspired by Go. This does not
  mean importing Go code: a Yxy package contains `.yxy` sources, and the path
  is only a naming and location scheme.
- **[AUTHOR-2] (author)** Compiled Yxy applications and Yxy users do not
  depend on Go or on the package service. The service (planned in Arandu, in
  Go) is separate infrastructure. The language, the compiler, the resolver
  and compiled programs never need it.

The rule this draft is built around is a proposal, not the author's decision
(it restates the bootstrap instructions, §14, and the review, PKG-01):

- **[UPD-1]** A normal build never updates versions silently. Adding or
  updating a dependency is an explicit operation whose result can be
  reviewed.

## 1. Terms

| Term | Meaning |
|---|---|
| package | The unit of import and of visibility: the `.yxy` files directly inside one directory of a module ([PKG-1]), or one standalone file ([PKG-7]) |
| module | The unit of versioning and distribution: a directory tree with `yxy.toml` at its root, containing packages in its root and subdirectories ([MOD-1]) |
| module path | The canonical path a module declares in its manifest; the prefix of the import path of each of its packages |
| import path | The path that names a package: the module path, followed by the package's directory relative to the module root |
| main module (project) | The module whose manifest and lock govern the command being run ([MOD-3]) |
| requirement | A minimum version of another module, declared in a manifest ([MAN-3]) |
| build list | The main module and the modules selected for it, one version each ([VER-5]) |
| dependency | A module in the build list other than the main module |
| local store | The per-user store of downloaded and verified module content (implementation decision 0004; review CACHE-02) |

### 1.1 Packages

- **[PKG-1]** Inside a module, a package is the set of `.yxy` files located
  directly in one directory. Each subdirectory is a different package. A
  directory with no `.yxy` file is not a package. Files whose name starts
  with `.` or `_` are not part of a package (as in Go; this also leaves out
  editor and AppleDouble files such as `._main.yxy`). The name of every other
  `.yxy` file is a valid path element ([PATH-2], [PATH-3]); another name, or
  a name that is not valid UTF-8, is an error, never skipped silently.
- **[PKG-2]** Directories whose name starts with `.` or `_`, and directories
  named `testdata`, are excluded: they are not packages and not part of any
  import path. Any other directory that contains `.yxy` files and whose name
  is not a valid path element ([PATH-2], [PATH-3]) is an error when the
  module is loaded; it is never skipped silently, and neither is a name that
  is not valid UTF-8. A subdirectory that has its own `yxy.toml` is another
  module and is not part of the enclosing one. The manifest is recognized
  only by its exact name, `yxy.toml`, whatever the case rules of the file
  system. Symbolic links inside a module are not followed: a symbolic link
  where a package directory or a source file is expected is an error, as in
  module content ([LOCK-6]), so a package has one location and one
  identity.
- **[PKG-3]** Every file of a package starts with the package clause
  `package <name>`, and all files of a package declare the same name. When
  the last element of the package's import path, ignoring a final
  major-version element ([VER-4]), is an identifier ([LEX-4]) that is not a
  keyword, a reserved word or a prelude name, `<name>` is that element.
  Otherwise `<name>` is any identifier, and importers must give an alias
  ([IMP-3]). The name is a checked label: the identity of a package is its
  import path ([PATH-1]), not its clause.
- **[PKG-4]** Item names are unique in the package, across all its files. An
  item declared in one file is visible, without `pub` and without an import,
  in every file of the same package.
- **[PKG-5]** The meaning of a package does not depend on the order of its
  files. Tools process files in the byte order of their names, so that
  diagnostics and outputs are deterministic.
- **[PKG-6]** Every `.yxy` file of the directory belongs to the package on
  every target. There are no conditional or per-target files in this version
  (OPEN #26).
- **[PKG-7]** A **standalone file** is a `.yxy` file with no `yxy.toml` in
  its directory or in any ancestor directory. It is a package of one file,
  with no import path, and may import only standard-library packages
  ([STD-1]). Today's single-file programs stay valid this way, and a
  directory of standalone programs, such as a test corpus, is not merged
  into one package.
- **[PKG-8]** A package that defines `fn main()` is an executable package.
  It cannot be imported.

**Why a package is a directory.** An origin path names a place in a
repository tree, and a directory is what a path naturally names, as in Go. A
library can then be split into files without making its internal helpers
public: files are organization, `pub` is interface. The cost is context: to
find an unqualified name, a reader may need the other files of the
directory. This draft limits that cost: imports are per file ([IMP-4]),
every name from another package is qualified ([IMP-6]), shadowing does not
exist ([DECL-3]), and tools can answer where a name is declared. The
alternative, one file per package as in [PRG-1] today, is recorded in
OPEN #15.

### 1.2 Modules

- **[MOD-1]** A module is the directory tree rooted at a directory that
  contains `yxy.toml`, minus nested modules. The manifest declares the module
  path. The package in the module root has the module path as its import
  path; a package in the subdirectory `a/b` has the import path
  `<module path>/a/b`.
- **[MOD-2]** The module is the unit that is required, selected, locked,
  fetched and verified — never a single package. A version of a module is an
  immutable set of files ([LOCK-5]).
- **[MOD-3]** The main module is the module that contains the packages a
  command operates on: the nearest directory, from their directory upwards,
  that contains `yxy.toml`. Only the main module's manifest and lock govern
  resolution. A dependency contributes only its module path, its minimum
  language version and its requirements ([MAN-6]); its lock is ignored.
- **[MOD-4]** A workspace of several local modules developed together is not
  defined in this version. Local development uses path replacements
  ([MAN-4]) in the main module (OPEN #29).

### 1.3 From `module <name>` to `package <name>`

Today every file starts with `module <name>`, and that file is the whole
program ([PRG-1]). In this draft "module" is the versioned unit declared in
`yxy.toml`, so keeping the clause would give one word two meanings: the
clause would say "module" where it declares a package.

- **[MIG-1]** In the version that implements packages, the clause is
  `package <name>` ([PKG-3]); `module` becomes a reserved word, and a file
  that starts with `module <name>` is rejected with a mechanical fix to
  `package <name>`, as `@io` is fixed to `@out` ([LEX-12]). There is no
  period in which both clauses are accepted: one textual form per concept.
- **[MIG-2]** Until then nothing changes: `module <name>` stays the clause
  and a program is one file. To keep the migration mechanical, `package`
  should be reserved now ([LEX-6]), so that no program uses it as a name.
  The compiler's corpus is rewritten in the same change that implements
  packages.

The alternative — keep `module <name>` and read it as the package name — is
recorded in OPEN #16.

## 2. Imports

```yxy
package inventory

import "github.com/example/collections"
import "github.com/example/yxy-bits/v2" as bits

pub fn aligned_free_slot(slots: &[u64]) -> Result<usize, collections.Error>
effects {}
{
    @ctrl:
        require slots.len > 0 else collections.Error.Empty

    <- @eval:
        index := collections.first_zero(slots)?
        aligned := bits.align_up(index, 8)

    -> @out:
        return Ok(aligned)
}
```

- **[IMP-1]** An import is `import "<import path>"` or
  `import "<import path>" as <name>`, one per line. There is no grouped
  form.
- **[IMP-2]** Imports follow the package clause and precede every item. An
  import after an item is an error.
- **[IMP-3]** An import binds exactly one name, its **local name**: the alias
  when there is one; otherwise the last element of the import path, ignoring
  a final major-version element ([VER-4]). When that element is not an
  identifier ([LEX-4]), or is a keyword, a reserved word or a prelude name,
  the alias is required. So the local name is known from the import line
  alone, without opening the imported package ([PKG-3] makes it agree with
  the package's own clause).
- **[IMP-4]** The local name belongs to the file that contains the import,
  not to the package: each file lists the imports it uses.
- **[IMP-5]** A local name cannot equal another local name of the same file,
  the name of an item of the package, or a prelude name ([PRG-2]); a
  parameter or local variable cannot take the name of an import of its file
  (extending [DECL-3]). A package's own name is not in scope: a package names
  its own items without qualification.
- **[IMP-6]** Names from another package are always qualified with the local
  name: a function `collections.first_zero(s)`, a type `collections.Error`,
  a variant `collections.Error.Empty` in expressions and in patterns. There
  are no glob imports and no imports of single items; `use` stays reserved.
  Only `pub` items can be named ([VIS-1]).
- **[IMP-7]** A local name is not a value. It can only be the first part of a
  qualified name; `x := collections` is an error.
- **[IMP-8]** Every import is used: an import whose local name does not
  appear in its file is an error, with a mechanical fix that removes the
  line. There are no imports for side effects, since importing has no effect
  ([INIT-1]); `import "…" as _` is an error.
- **[IMP-9]** Importing the same path twice in one file is an error, also
  under different aliases; so are two imports with the same local name.
- **[IMP-10]** The import graph of packages is acyclic. An import that closes
  a cycle is an error whose diagnostic lists the whole cycle, each edge with
  the file and line of its import. A package that imports its own path is a
  self-import, reported as such.
- **[IMP-11]** Import paths are absolute and canonical ([PATH-1]). A package
  of the same module is imported by its full path; relative paths such as
  `"./util"` or `"../x"` are errors. One package has exactly one spelling.
- **[IMP-12]** An import path must belong to the standard library ([STD-1]),
  to the importing package's own module, or to a module that this module
  requires **directly** in its manifest ([MAN-3]); a module reached only
  through another dependency cannot be imported. The providing module is the
  module of the build list whose path is the longest element-wise prefix of
  the import path; when two modules could provide the same package path, the
  import is ambiguous and an error. An import of a module that is not
  required is an error whose note shows the explicit command that would add
  it ([DEP-4]). The compiler never guesses or substitutes a module, and
  downloads only content the lock pins ([DEP-9]; [UPD-1], [NET-1]).
- **[IMP-13]** An import names a package. A path with no package — for
  example a module root without `.yxy` files — is an error.
- **[IMP-14]** *(tooling)* `yxy fmt` writes imports one per line, sorted by
  the bytes of their paths, as one block after the package clause. It never
  adds or removes an import.

## 3. Visibility

- **[VIS-1]** An item is private to its package unless its declaration starts
  with `pub`. `pub` applies to `fn` (including `extern fn` and `export fn`)
  and `enum`, and to `struct` when structs exist. Other packages can name
  only `pub` items.
- **[VIS-2]** `pub enum` makes the type and all its variants visible;
  variants have no visibility of their own. Proposal for structs (OPEN #28):
  the fields of a `pub struct` are private unless each is marked `pub`, so
  that a package can keep the invariants of its values.
- **[VIS-3]** A public signature names only public types: every parameter
  and return type of a `pub fn` is a prelude type or a `pub` type. A private
  type in a public signature is an error.
- **[VIS-4]** `fn main` cannot be `pub` ([PKG-8]).
- **[VIS-5]** Effects are part of every public signature. The effects of an
  imported function are the ones its signature declares, and a call across
  packages is checked by [EFF-3] exactly like a call inside a package; the
  caller never depends on the callee's body. Therefore adding an effect to a
  `pub fn` is an incompatible change ([VER-3]); removing one is compatible.
- **[VIS-6]** The foreign-code trust boundary propagates. A `pub fn` that
  calls foreign code declares `ffi` ([EFF-3], [EFF-4]), so every caller, in
  every package, declares it too, and the guarantees of [EFF-5] keep their
  scope across packages. *(tooling)* Tools report, per module of the build
  list, whether it declares `extern fn` or `export fn`, and the diff of an
  add or update ([DEP-3]) shows when a module with foreign declarations
  enters the build.
- **[VIS-7]** A named type (an enum, later a struct) is identified by the
  import path of its package and its name. Two packages that declare
  `enum Error` declare two distinct types. Two major versions of a module are
  two modules ([VER-4]), so their types are distinct too.
- **[VIS-8]** `export fn` names are global to the linked program: two
  `export fn` with the same name anywhere in the program are an error before
  linking. Two `extern fn` declarations of the same C symbol in different
  packages must have identical parameter types, return type and effects;
  otherwise they are an error. Functions that are not `export` never clash
  across packages: their symbols derive from the package's identity (the
  encoding is an implementation matter). The reserved names of [ABI-3] apply
  in every package.

## 4. Identity

### 4.1 Paths

- **[PATH-1]** A module path or import path is a sequence of one or more
  elements separated by `/`, with no leading or trailing `/` and no empty
  element. Paths are compared byte by byte; the spelling in the source is
  the canonical form.
- **[PATH-2]** Paths are ASCII and use only lowercase letters `a`–`z`,
  digits, `-`, `.` and `_`, besides the `/` separator. Uppercase letters and
  every non-ASCII character are errors. Hence no case folding and no Unicode
  normalization is ever applied: two different paths differ in their bytes on
  every file system, case-insensitive ones included, and no confusable
  spelling of a path can exist (compare [LEX-3a], [LEX-4]; OPEN #31).
- **[PATH-3]** A path element starts with a letter or a digit, does not end
  with `.`, and, before its first `.`, is not a name reserved by Windows
  (`con`, `prn`, `aux`, `nul`, `com1`–`com9`, `lpt1`–`lpt9`).
- **[PATH-4]** The first element is either a standard-library root
  ([STD-1]) or an **origin**: a lowercase DNS name with at least one dot,
  such as `github.com` or `git.example.org`. Every first element without a
  dot is reserved for the language.
- **[PATH-5]** A path has at most 32 elements and 256 bytes. A longer path
  is an error, never truncated.
- **[PATH-6]** A path names the origin a module is expected to come from.
  It is not a claim about the author, and by itself proves nothing about
  content; the lock does ([LOCK-4]). A module obtained for a path must
  declare that same module path in its own manifest; a mismatch is an error.
- **[PATH-7]** The location from which a module's versions are obtained is
  derived from its path for repository hosts whose repository root is known
  (for `github.com/example/collections`, the repository
  `github.com/example/collections`), or named explicitly in the main
  manifest ([MAN-5]). A module may live in a subdirectory of its repository.
  How the repository root is discovered for other origins is open
  (OPEN #22).
- **[PATH-8]** An alias ([IMP-3]) is a local name only. It never changes the
  identity of a package, of its types or of its lock entry. A module used
  from a local directory ([MAN-4]) also keeps its module path as identity:
  the directory says where its content is, not what it is.

### 4.2 The standard library

- **[STD-1]** `core`, `alloc` and `std` are the roots of the standard
  library (`import "core/…"`, `"alloc/…"`, `"std/…"`). It is supplied by the
  toolchain, at the toolchain's version, and is never required in a
  manifest, fetched, replaced or locked (OPEN #30).
- **[STD-2]** Layering follows the author's bootstrap instructions (§15):
  `core` needs neither allocation nor an operating system; `alloc` adds what
  needs an allocator and imports only `core`; the hosted `std` may import
  `core` and `alloc`. A freestanding program imports only `core`, and
  `alloc` when it supplies an allocator; the profiles themselves are future
  work.
- **[STD-3]** No import is magic or global. Nothing is imported implicitly,
  no package adds names to another, and a program uses a standard-library
  package only by importing it. The prelude ([PRG-2]) is the fixed, closed
  list of names known to the compiler: it is not a package, and neither the
  standard library nor a dependency can extend it.
- **[STD-4]** No module outside the toolchain may declare a path under a
  reserved root, and no replacement may target one ([MAN-4]). The standard
  library does not exist yet, so every standard import is an error in this
  version.

## 5. Nothing runs at import or resolution

- **[INIT-1]** A package contains only declarations. There are no package
  initializers, no top-level statements, no global variables initialized at
  start-up and no `init` functions. Importing a package has no effect at run
  time: the only code a program runs is reached from `main`, or from an
  `export fn` called by foreign code.
- **[INIT-2]** Compile-time constants, if they are added, are evaluated by
  the compiler under explicit limits (as [GR-5] limits parsing), have no
  effects, and do not run package code during resolution.
- **[INIT-3]** There are no build scripts, install hooks, code generators or
  compiler plug-ins run on behalf of a package. Obtaining, verifying,
  selecting and building a module never executes code supplied by that
  module or its origin — neither Yxy code nor programs, nor version-control
  features that run code from the repository (hooks, filters, submodule
  commands).
- **[INIT-4]** C sources or native libraries needed by a dependency are not
  supported in this version. Linking foreign code stays an explicit option
  of the main build (OPEN #27).

## 6. Manifest and lock — proposal

The names come from the author's bootstrap instructions (§14): `yxy.toml`
declares requirements; `yxy.lock` records the exact resolution. What follows
is the **contract** — what each file contains and who writes it. The TOML
layouts are illustrative, not final (OPEN #17).

### 6.1 Manifest: `yxy.toml`

- **[MAN-1]** The manifest is written by people and by the explicit
  dependency commands ([DEP-1]). It contains the module path, the minimum
  language version ([MAN-7]), the requirements ([MAN-3]) and — meaningful
  only in the main module — replacements ([MAN-4]), private path patterns
  ([NET-3]) and explicit sources ([MAN-5]); and optionally the module's
  declared license expression ([LOCK-2] item 6).
- **[MAN-2]** A manifest never contains credentials, tokens or
  authenticated URLs; they belong to the user's configuration, outside the
  project.
- **[MAN-3]** A requirement names a module path and one exact version: the
  minimum this module needs ([VER-5]). There are no ranges, `latest`, branch
  names or wildcards. A module lists every module whose packages it imports
  ([IMP-12]).
- **[MAN-4]** The main module may **replace** a module path, for every
  version, by a local directory (development, a local fork) or by another
  module path at an exact version (a published fork). A replacement is
  explicit, written in the main manifest, recorded in the lock ([LOCK-2]) and
  shown in every add or update diff ([DEP-3]). The replaced module keeps its
  import path and identity in the source ([PATH-8]); only its content
  changes. A replacement declared by a dependency is **not applied** — never
  inherited — and the add and update commands report it as a note. A module
  that was never published is required at version `0.0.0` and replaced by
  its directory; a shorter spelling is open (OPEN #17).
- **[MAN-5]** The main manifest may name the source of a module path (a
  repository location) when it cannot be derived from the path ([PATH-7]).
  A source says where bytes come from, never what they are: the content is
  verified against the lock whatever the source.
- **[MAN-6]** A dependency's manifest contributes only its module path, its
  minimum language version and its requirements. Its replacements, private
  patterns and sources concern its own development and are ignored.
- **[MAN-7]** The minimum language version is checked, not acted on: a
  toolchain older than a module's minimum refuses that module with a
  diagnostic. No toolchain is downloaded or switched automatically.

```toml
# yxy.toml (layout of compiler implementation decision 0006; OPEN #17)
[module]
path = "github.com/example/inventory"
yxy = "0.1"

[require]
"github.com/example/collections" = "1.4.2"
"github.com/example/yxy-bits/v2" = "2.0.1"

[replace]
"github.com/example/collections" = { dir = "../collections" }

[private]
paths = ["git.corp.example/*"]
```

### 6.2 Lock: `yxy.lock`

- **[LOCK-1]** The lock is written only by the explicit dependency commands
  ([DEP-1]), never by a build and never by hand. It is kept under version
  control next to the manifest. Only the main module's lock is used.
- **[LOCK-2]** The lock contains its format version; the identity of the
  resolver that selected its build list; the requirements and replacements it
  was computed from; and, for **every** module of the build list — the whole
  transitive closure ([VER-5]):
  1. the canonical module path and the selected version;
  2. the origin revision: the version-control system, the hash algorithm and
     the full identifier (for Git, `sha1` or `sha256` and the complete object
     id, never an abbreviation);
  3. the content hash: its algorithm and digest over the module's content
     ([LOCK-3]);
  4. the hash of its manifest and the requirements it contributes;
  5. the replacement applied to it, if any: for a local directory, the
     relative path and **no** content hash, marking the entry as live local
     content that the lock does not reproduce; for a published fork, the
     fork's path, version, revision and content hash;
  6. declared metadata such as the license expression, recorded as declared,
     which is not a legal audit.

  It also records, for versions read during selection but not selected,
  their manifest hash and their requirements, so that the selection can be
  recomputed from the lock alone, without the network — with the resolver
  the lock names. **(author, 2026-09-26, Q2)** The resolver identity is
  versioned: a lock whose resolver the tool does not implement is refused,
  never read as if another resolver had selected it, and a new selection
  algorithm gets a new identity (compiler: `resolver = "mvs-1"` for the
  minimal version selection of [VER-5]; implementation decisions 0006 and
  0008).
- **[LOCK-3]** A module's content is every regular file under its root,
  excluding nested modules and version-control metadata, each identified by
  its path relative to the root; Git submodules are not content. The content
  hash covers a canonical, sorted encoding of paths and file digests,
  independent of archive format, timestamps and file permissions. The
  encoding is `yxy-content-v1` (compiler implementation decision 0006):
  `sha256` over `yxy-content-v1\n` followed by one `<hex sha256>  <path>\n`
  line per file, sorted by path bytes. Every hash names its algorithm (for
  example `sha256:`), so that the algorithm can be replaced. The encoding is
  fixed by the lock's format: lock `format = 1` uses `yxy-content-v1`, and a
  new encoding requires a new `format` (the value names the algorithm, not
  the encoding; whether it should also name the encoding is OPEN #17).
- **[LOCK-4]** A content hash proves that the bytes in use are the bytes
  approved when the lock was written. It does not prove that the code is
  safe, correct, licensed as declared, or written by whoever the origin
  claims. A checksum proves identity against a trusted reference, not
  safety.
- **[LOCK-5]** A module version is immutable. When an origin serves other
  content or another revision for a locked version — a moved tag, rewritten
  history, a corrupted archive — the tool refuses it with an integrity
  error. It never replaces approved content silently and never "repairs"
  the lock by itself.
- **[LOCK-6]** Before content enters the local store, the tool limits its
  expanded size and number of files; rejects absolute paths, `..`, symbolic
  links and special files; and rejects two paths that are equal under case
  folding or Unicode normalization, so that a module has the same files on
  every operating system. The limits' values are open (OPEN #17). A rejected
  module is rejected as a whole.
- **[LOCK-7]** The lock is independent of the target ([BLD-1]).

```toml
# yxy.lock (generated by the dependency commands; do not edit; illustrative)
format = 1
resolver = "mvs-1"

[[module]]
path = "github.com/example/yxy-bits/v2"
version = "2.0.1"
revision = { vcs = "git", algorithm = "sha1", id = "<40 hexadecimal digits>" }
content = "sha256:<64 hexadecimal digits>"
manifest = "sha256:<64 hexadecimal digits>"
requires = ["github.com/example/collections 1.2.0"]
license = "BSD-3-Clause"   # declared by the module, not audited
```

### 6.3 Operations and modes

- **[DEP-1]** Only explicit commands change the manifest or the lock.
  Proposed names, not final: `yxy add <path>[@<version>]`,
  `yxy update [<path>[@<version>]]`, `yxy remove <path>`, and `yxy lock`,
  which recomputes the lock after a manual edit of the manifest. `yxy fetch`
  obtains the content the lock pins without building ([DEP-9]) and changes
  neither file.
- **[DEP-2]** The same manifest and the same origin content produce the same
  lock, byte for byte. Because selection uses minimum versions ([VER-5]),
  `yxy lock` never picks a version newer than some requirement names, and
  needs no list of available versions. Only `add` or `update` without an
  explicit version asks an origin which versions exist.
- **[DEP-3]** Every change is reviewable: `add`, `update`, `remove` and
  `lock` print the diff of the manifest and of the build list — modules
  added, removed, upgraded or downgraded, with revisions and content hashes;
  replacements; replacements of dependencies that are not applied; modules
  that bring foreign declarations ([VIS-6]). With `--dry-run` they print the
  diff and write nothing.
- **[DEP-4]** A tool never adds a dependency that the user did not name: no
  dependency is inferred from an import, guessed from a similar name or
  suggested silently by an assistant. An import of a module that is not
  required is an error ([IMP-12]) whose note shows the exact command; running
  it is the user's decision.
- **[DEP-5]** Builds — every command that loads a module's packages:
  `check`, `build`, `run`, `test`, `inspect` and the reference evaluator
  (`dev eval`) — never write the manifest or the lock and never select
  versions: they use the lock exactly. A lock that does not match the manifest — a changed
  requirement or replacement, a changed manifest of a module replaced by a
  local directory, an imported module that is not locked — is an error that
  names the command to run.
- **[DEP-6]** **Locked** mode (`--locked`): no command writes the manifest or
  the lock; a command that would change either fails and prints the diff it
  would have made. Builds are always locked ([DEP-5]); the flag turns `add`,
  `update`, `remove` and `lock` into checks, for continuous integration.
- **[DEP-7]** **Offline** mode (`--offline`): no network request of any
  kind — no name lookup, origin, mirror or index. Only the local store is
  used; missing content is a diagnostic that names the module, the version
  and the content hash, and points to `yxy fetch` to prepare offline work.
- **[DEP-8]** Content enters the local store only after it has been written
  completely and verified against the lock or, during `add` and `update`,
  against the reference in force ([NET-5]). A partial or unverified entry is
  never used (review CACHE-03). Projects share verified content read-only;
  a build never modifies stored content (review CACHE-02).
- **[DEP-9]** **(author, 2026-09-25)** A build may obtain content that the
  lock pins and that is missing from the local store, as `yxy fetch` does:
  exactly the locked revision of each locked module, verified against the
  locked content hash before any use ([DEP-8]), and reported, one line per
  module obtained. A build never selects a version, never requests a module
  or version that the lock does not pin, never writes the manifest or the
  lock ([DEP-5]), and never falls back to another source ([NET-4]). A
  verification failure stops the build with an integrity error ([LOCK-5]).
  With `--offline` a build makes no request at all ([DEP-7]).

### 6.4 Network, privacy and trust

- **[NET-1]** Network access happens only in the commands of [DEP-1] and,
  to obtain content the lock pins, in builds ([DEP-9]) — never in `--offline`
  mode, never to choose what to use, and never at run time of a compiled
  program ([AUTHOR-2]).
- **[NET-2]** The toolchain obtains modules directly from their origins, or
  from local directories, without any package service. A mirror, proxy or
  catalog, such as the planned service, is optional and configured
  explicitly. A catalog or search index is never an authority for
  resolution. The service's protocol is Yxy's own; being inspired by Go's
  module proxy does not make it compatible with Go's proxy, `go.mod` or
  checksum database (OPEN #22, #23).
- **[NET-3]** Private origins: the main manifest and the user's
  configuration list path patterns of private modules, and the policy is
  applied **before the first network request** for any path. A private path
  is requested only from its own origin, or from a private source configured
  for it, with the user's credentials. It is never sent to a public mirror,
  index, catalog or checksum service, not even as a lookup (OPEN #21).
- **[NET-4]** An authentication, authorization or integrity failure stops
  the operation. There is no fallback to another source, and in particular
  to a less trusted one, unless that fallback is configured explicitly for
  that path; private paths never fall back to public sources.
- **[NET-5]** When a module version enters a project for the first time
  (`add` or `update`), its revision and hashes are computed from the content
  obtained and become the reference when the user accepts the diff; from then
  on, the lock is the trusted reference. An independent reference for first
  acquisitions — a checksum log or signatures — is open (OPEN #20).
- **[NET-6]** Diagnostics, logs and reports never contain credentials,
  tokens or authenticated URLs. The toolchain sends no telemetry.

## 7. Versions

- **[VER-1]** A version is `MAJOR.MINOR.PATCH` with an optional pre-release
  (`2.0.0-rc.1`), ordered by the precedence of Semantic Versioning 2.0.0.
  Leading zeros and build metadata (`+…`) are not allowed.
- **[VER-2]** A version exists in the origin as a tag on a revision: `v1.4.2`
  for a module at the repository root, `<subdirectory>/v1.4.2` for a module
  in a subdirectory, as in Go. The lock pins the revision and the content,
  so a tag that moves later is detected ([LOCK-5]). An untagged revision
  cannot be required in this version; a replacement serves that need
  (OPEN #25).
- **[VER-3]** Within one major version a later version is expected to be
  compatible: it keeps every `pub` item with its types and adds no effect to
  an existing `pub fn` ([VIS-5]). This is a promise of the module's authors,
  as in Go's import compatibility rule; nothing verifies it yet, and a checker
  of public interfaces is future tooling. Versions `0.x.y` promise nothing.
- **[VER-4]** Major versions coexist. From major 2 on, the module path ends
  with the element `v<N>` (`github.com/example/yxy-bits/v2`), with no leading
  zero; majors 0 and 1 have no such element, so they share one module path
  and are **one module**: versions `0.x.y` and `1.x.y` of it are ordered and
  selected together ([VER-5]), and a build may move from 0 to 1 like from one
  minor to the next. Every other major is a different module, with its own
  import path, packages and types ([VIS-7]); one build may use several. That
  element is ignored when deriving a local name ([IMP-3]). Provisional
  (OPEN #18; for `0.x`, OPEN #19).
- **[VER-5]** Selection is **minimal version selection** (experimental;
  OPEN #19). Each module version requires minimum versions of other modules.
  Walking these requirements from the main module, after replacements, the
  build list keeps for each module path the highest version reached. Hence:
  the result is deterministic; publishing a new version never changes it, and
  only an explicit update does ([UPD-1]); no constraint solver is needed;
  and the lock can be re-verified from recorded manifests. There is no
  version conflict to resolve: within one module path (one major, or majors
  0 and 1 together, [VER-4]) the highest minimum wins, and the other majors
  are different modules. Requirements between modules may
  form cycles; only the package import graph must be acyclic ([IMP-10]).
  Accepted limits: requirements have no upper bounds or exclusions, and a
  release that breaks compatibility within a major breaks its dependents
  until it is fixed or replaced.
- **[VER-6]** Downgrades and removals are explicit (`update <path>@<older>`,
  `remove`) and follow the same selection. Replacements rewrite the
  requirement graph before selection.

**Why minimal version selection now.** It is the only candidate examined that
gives [UPD-1] by construction instead of by discipline: the selection is a
function of the manifests, not of what an origin published last. It has been
used by Go since modules were introduced. With it, the lock's role is content
identity — revisions, hashes and the closure a build needs offline — and a
reviewable record, not the only barrier against silent upgrades. It stays
open because it relies on a compatibility promise that nothing checks yet.

## 8. Targets

- **[BLD-1]** Source identity is independent of the target: import paths,
  versions, revisions, content hashes, the build list and the lock are the
  same for every target ([TGT-1]; decision 0010). Resolution never looks at
  the target.
- **[BLD-2]** Validity per target is unchanged: a package is valid for a
  target by the rules of the specification ([TGT-2]). The same locked source
  may compile for a 64-bit target and be rejected for a 32-bit one.
- **[BLD-3]** Build artifacts are specific to the target: analysis results,
  objects and executables derived from a package are reused only when every
  input matches, the complete target included (review CACHE-02;
  implementation decision 0004, item 2). Reusing a package's source never
  implies reusing its binaries.
- **[BLD-4]** There are no target-conditional files or requirements in this
  version ([PKG-6]; OPEN #26).

## 9. Proposed grammar additions

Integrated into `syntax.md` §3, where the first rule replaced
`module = "module" IDENT NL { item } ;`.

```ebnf
file        = package_clause { import_decl } { item } ;
package_clause
            = "package" IDENT NL ;
import_decl = "import" PATH [ "as" IDENT ] NL ;
item        = [ "pub" ] ( enum_decl | struct_decl | fn_decl ) ;
field       = [ "pub" ] IDENT ":" type ;       (* in struct_decl; [VIS-2], OPEN #28 *)

qual_name   = IDENT [ "." IDENT ] ;             (* item, or import.item *)

type        = qual_name [ "<" type { "," type } ">" ]
            | "[" type ";" INT "]"
            | "&" "[" type "]"
            | "(" ")" ;

pattern     = "_" | "true" | "false" | [ "-" ] INT | "(" ")"
            | IDENT                             (* binding, or `None` *)
            | IDENT "(" pattern ")"             (* Some, Ok, Err *)
            | IDENT "." IDENT                   (* Enum.Variant *)
            | IDENT "." IDENT "." IDENT ;       (* import.Enum.Variant *)
```

Lexical additions:

```ebnf
PATH      = '"' path_char { path_char } '"' ;
path_char = "a"…"z" | "0"…"9" | "-" | "." | "_" | "/" ;
```

- A quoted literal is valid only as the path of an import: on one line,
  without escape sequences. A quoted literal anywhere else remains an
  unsupported string literal ([LEX-11]). The lexer reads any printable
  ASCII between the quotes, and [PATH-1]–[PATH-5] are then checked, so that
  an uppercase letter gets a specific diagnostic rather than a lexical one.
- `package`, `import`, `pub` and `as` become keywords; `module` becomes a
  reserved word ([MIG-1]); `use` stays reserved.
- Expressions keep their grammar. `collections.first_zero(x)` parses as a
  name, `.first_zero` and a call; [GR-1] is extended so that the callee is a
  name or a qualified name whose first part is an import ([IMP-6]). Since a
  local variable can never share a name with an import ([IMP-5]), `a.b` is
  never ambiguous between a field or `.len` and a qualified name.
- `NL` handling is unchanged: the package clause and each import end at
  their line break ([NL-2], [NL-6]).

## 10. Out of scope, and the compiler meanwhile

### Implementation status

The status of each rule, with its tests, is in the compiler repository
(`docs/implementation/STATUS.md`, requirements R92–R109 and R200–R214, and
implementation decisions 0006, 0007 and 0008). In short, as of 2026-09-27:

- **Implemented** for packages of one module: [MIG-1] (the clause is
  `package`; `module` is rejected with a mechanical fix; [MIG-2] is past),
  [PKG-1]–[PKG-8] (with the file names and links of [PKG-1] and [PKG-2]),
  [MOD-1], [MOD-3], [MAN-7] (this toolchain implements language 0.1), [IMP-1]–[IMP-11],
  [IMP-13], [IMP-14], [VIS-1]–[VIS-5], [VIS-7], [VIS-8], [PATH-1]–[PATH-5],
  [STD-1], [STD-3], [STD-4], [INIT-1], the grammar of §9, and the tooling
  part of [VIS-6] (facts say whether a package declares foreign functions).
  The provisional field rule of OPEN #28 is implemented: fields are private
  to their package unless declared `pub`.
- **Implemented** for dependencies (implementation decision 0008): [IMP-12]
  (a module imports the packages of the modules it requires directly; the
  note of an import that is not required names the manual step, `[require]`
  and `yxy lock`, [DEP-4]), [MOD-2], [PATH-6], [PATH-7] (a `[source]` of the
  main manifest, or the repository derived for `github.com` and
  `codeberg.org`), [PATH-8], [MAN-1]–[MAN-7], [LOCK-1]–[LOCK-7] (with the
  resolver identity of [LOCK-2], `resolver = "mvs-1"`), [DEP-2], [DEP-4],
  [DEP-5] (except `test`, which does not exist yet: its part of [DEP-5] is
  due with the command), [DEP-6] and [DEP-7] for `yxy lock`, [DEP-8],
  [DEP-9], [VER-1], [VER-2], [VER-4], [VER-5], [NET-1]–[NET-4], [NET-6],
  [INIT-3]; `yxy lock` and `yxy fetch`. The paths of modules and imports are
  checked by one validator of [PATH-1]–[PATH-5]. `yxy lock` trusts the previous
  lock for identity only (the revision and content hash it pins for a
  version, [LOCK-5]), and reads every fact from the content at each version's
  tag ([DEP-2]): a moved tag of a pinned version is refused, never rewritten;
  `yxy fetch` prepares `yxy lock --offline` ([DEP-7]); the "local store" of [DEP-7] is the toolchain's home, its
  content store and its repository cache; a nested module or repository is
  not content, nor a package, by one rule ([PKG-2], [LOCK-3]); and a path of
  content must also fit under the local store on the platform ([LOCK-6];
  OPEN #17).
- **Partly**: [DEP-1] and [DEP-3] (`yxy lock` and `yxy fetch` only: `add`,
  `update` and `remove` do not exist yet and keep failing, compiler
  requirement R38; the review of `yxy lock` lists the changes of the build
  list, with why each module is at its version, but not the foreign
  declarations of [VIS-6]); [VER-6] (a downgrade by editing the requirement
  and running `yxy lock`; no `update <path>@<older>`); [NET-5] (the lock is
  the reference; no checksum log). [MOD-4]: no workspaces.
- **Not implemented**: §8 (targets of the build list), [STD-2], the package
  service, and the `--dry-run` of [DEP-3].

Outside this draft, for later designs: the package service and its protocol;
signatures and checksum logs; publishing; retractions and exclusions;
workspaces; target-conditional code; native code in dependencies; test
packages; documentation generation; re-exports and module-internal packages;
the contents and profiles of the standard library; binary caches.

Suggested order of implementation (not normative), local before network as
the review recommends: (1) multi-file packages, `pub`, qualified names and
imports inside one module, with standalone files unchanged; (2) the manifest,
path replacements and the lock, with no network; (3) `fetch` from origins
with verification, tested against local repositories; (4) `add`, `update`
and `remove`; (5) the optional service.

Conformance tests to add with the implementation: one rejection program per
error rule ([PKG-2], [PKG-3], [PKG-8], [IMP-2], [IMP-3], [IMP-5], [IMP-7] to
[IMP-13], [VIS-1], [VIS-3], [VIS-4], [VIS-8], [PATH-2] to [PATH-6],
[STD-4]); and the acceptance gates of the review (tampered package, moved
tag, offline build with pinned dependencies, private project with zero
public requests, and add or update producing the same resolution from the
same inputs).

## Appendix: differences from Go (informative)

| Topic | Go | This draft |
|---|---|---|
| Letters in paths | uppercase allowed, escaped in the module cache | lowercase ASCII only ([PATH-2]) |
| Package name and path | independent | equal to the last element when that is an identifier ([PKG-3]) |
| Requirements and hashes | `go.mod` and `go.sum` | `yxy.toml` and `yxy.lock`, which holds the whole closure ([LOCK-2]) |
| Builds and the network | a build may download required modules missing from the module cache | a build may download only what the lock pins, verified against the lock's content hash; `--offline` forbids any request ([DEP-9], [DEP-7]) |
| Replacements in dependencies | ignored | not applied, and reported in the add/update diff ([MAN-4]) |
| Toolchain version | since Go 1.21, a newer `go` line can make the `go` command download and run a newer toolchain | minimum checked, never downloaded ([MAN-7]) |
| Mirror and checksum database | a public proxy and checksum database by default | none by default; the service is optional and not Go-compatible ([NET-2]) |
| Package initialization | package variables and `init` functions | none ([INIT-1]) |
| Module-internal packages | `internal/` directories | not defined (OPEN #28) |

## Sources

Consulted on 2026-09-25; no text is copied.

- Go Modules Reference — https://go.dev/ref/mod (modules and packages,
  module path rules, major version suffixes and the import compatibility
  rule, `replace` applying only in the main module, `go.sum`, private
  modules, read-only `go.mod` by default since Go 1.16, the proxy protocol,
  minimal version selection).
- Go toolchains — https://go.dev/doc/toolchain (automatic toolchain
  selection and download since Go 1.21).
- Russ Cox, "Minimal Version Selection" —
  https://research.swtch.com/vgo-mvs (the algorithm; selection unaffected by
  new releases; no constraint solving; limits of minimum-only requirements).
- Semantic Versioning 2.0.0 — https://semver.org/spec/v2.0.0.html
  (precedence, pre-releases, build metadata, major version zero).
- The review that motivated this draft, PKG-01 to PKG-03 and CACHE-01 to
  CACHE-04: `plans/reviews/2026-09-25-cache-and-packages-review.md`.
- The author's bootstrap instructions, §7 (names, modules, imports), §14
  (manifest, lock, no scripts, no silent downloads, no guessed dependencies)
  and §15 (`core`, `alloc`, `std`).
