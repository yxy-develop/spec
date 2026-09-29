# Decision 0011: Packages, modules and imports by origin

- Status: Author (items 1–2 and 14), Experimental (items 3–13) — plans/decisions/README.md, L-0011.
  Items 1–2 are the author's direction (2026-09-25, given in conversation and
  recorded in the control repository as requirements C-2 and C-4); item 14
  was decided by the author (2026-09-25, the answer to A9 of the control
  repository); items 3–13 are open to the author's review. Packages of one
  module are implemented (compiler implementation decision 0007);
  dependencies from local directories and Git origins, `yxy lock`, `yxy
  fetch`, `--locked` and `--offline` too (compiler implementation decision
  0008, 2026-09-27); `add`, `update`, `remove` and the package service are
  not.
- Date: 2026-09-25
- Spec: `modules.md`. When integrated, it changes `semantics.md` [PRG-1],
  [PRG-2], [DECL-3], [EFF-3], [ABI-1], §12 and `syntax.md` [LEX-5], [LEX-6],
  [LEX-8], [GR-1], §3.
- Errata (2026-09-27): wording only; no rule changed. `modules.md` [VER-4]
  and [VER-5] now say explicitly what item 11 implied: majors 0 and 1 have no
  `/vN` element, so they share one module path and are one module, selected
  together; the questions of `0.x` versions and of modules reached only
  through requirements are recorded in `OPEN.md` #19 (architecture audit of
  2026-09-26, §15, C15a).
- Amendment (2026-09-27), **author's decision Q2** (2026-09-26, with the
  reviewer's amendment 4, both recorded in the control repository): the
  lock records the identity of the resolver that selected its build list,
  so that a future change of the selection algorithm never pretends to be
  compatible with an old lock; minimal version selection (item 11) stays
  experimental, and its comparison with the alternatives comes before the
  first public release. Text: `modules.md` [LOCK-2] (the identity, and a
  lock of an unknown resolver refused) and the illustrative lock of §6.2.
  The name and form of the field (`resolver = "mvs-1"`), its relation to
  `format`, and a lock without it are the compiler's implementation detail
  (implementation decisions 0006 and 0008; experimental).
- Errata (2026-09-27): wording only; no rule changed. [LOCK-3] states that
  the lock format fixes the content-hash encoding (format 1 uses
  `yxy-content-v1`; a new encoding needs a new format), which implementation
  decision 0006 implied (architecture audit of 2026-09-26, §15, e3); [DEP-5]
  names the reference evaluator (`dev eval`) among the commands that load a
  module's packages and so use the lock exactly, which the implementation
  already did (the same audit, "`yxy test`", P4).

## Context

In slice 1 a program is one file, which is one module ([PRG-1]); `import` and
`pub` are reserved and rejected. The author's bootstrap instructions ask for
scopes, privacy, imports and module cycles to be defined without magic global
imports, module initializers with arbitrary effects or automatic fetching of
unknown dependencies (§7); for a manifest `yxy.toml` and a lock `yxy.lock`,
local packages first, no arbitrary build scripts, no silent downloads and no
dependencies guessed by an assistant (§14); and for a `core`/`alloc`/`std`
layering (§15). Per their §2, those points are guidelines to validate, not
decisions to attribute to the author.

On 2026-09-25 the author gave a direction in conversation, recorded by the
review `plans/reviews/2026-09-25-cache-and-packages-review.md` (section
"Author direction"): import packages by a path tied to their origin, as in
Go; build the first package service in Arandu, in Go, as separate
infrastructure; keep compiled programs and users independent of it. The same
review proposes, without deciding (PKG-01 to PKG-03): separate package and
module; separate requirements from the exact resolution; forbid silent
updates; define compatibility, cycles, major versions and conflicts before
transitive resolution; make replacements explicit; version the protocol;
apply privacy before the first request; never treat a checksum as proof of
safety; reuse Go components only where their contracts match.

## Decision

Author's direction:

1. Packages are imported by a path associated with their origin, inspired by
   Go. This does not mean importing Go code.
2. Compiled Yxy applications and Yxy users do not depend on Go or on the
   package service, which is separate infrastructure.

Experimental (`modules.md`):

3. **No silent updates** ([UPD-1]): a normal build never updates versions;
   adding or updating a dependency is an explicit, reviewable operation.
   Restates the bootstrap instructions (§14) and the review (PKG-01).
4. **Package = directory** of `.yxy` files inside a module; a module is a
   tree rooted at `yxy.toml`; a standalone file outside any module is a
   one-file package that may import only the standard library, so today's
   programs stay valid. An executable package cannot be imported.
5. **`package <name>` replaces `module <name>`** when packages are
   implemented, with a mechanical fix and no period accepting both;
   `package` is reserved now.
6. **Imports**: `import "path"` with an optional `as` alias, one per line,
   after the clause and before items; the local name is derivable from the
   line; imports are per file; names from other packages are always
   qualified; no glob or single-item imports; unused, duplicate, cyclic,
   self and relative imports are errors; imports of modules that are not
   directly required are errors, never fetches.
7. **Visibility**: private by default, `pub` on items; public signatures use
   only public types; effects are part of public signatures, checked across
   packages by [EFF-3], so adding an effect is an incompatible change; type
   identity is the package path and the name; `export` symbols are unique in
   the program.
8. **Identity**: canonical, ASCII, lowercase paths compared byte by byte, so
   no case folding or Unicode normalization exists; origins are DNS names;
   `core`, `alloc` and `std` are reserved roots supplied by the toolchain;
   nothing is imported implicitly; the prelude is closed.
9. **Nothing runs** at import, resolution or build on behalf of a package:
   no initializers, build scripts, hooks or plug-ins.
10. **Manifest and lock contract**: requirements in `yxy.toml`; the exact
    closure in `yxy.lock` (path, version, full revision with algorithm,
    algorithm-tagged content hash, manifest hash, requirements, replacement,
    declared license). Only explicit commands write them, with a reviewable
    diff; builds are always locked; `--locked` and `--offline` for the
    dependency commands; replacements only from the main module,
    recorded, never inherited; private patterns applied before the first
    request; no fallback after authentication or integrity failures;
    checksums prove identity, not safety.
11. **Versions**: Semantic Versioning precedence; tags in the origin;
    majors 2 and above carry a `/vN` path element and coexist as distinct
    modules; selection by **minimal version selection**, provisional.
12. **Targets**: source identity and the lock are target-independent; build
    artifacts are target-specific (review CACHE-02).
13. **Grammar** (`modules.md` §9): `package`, `import`, `pub`, `as` become
    keywords, `module` reserved; import paths are quoted literals valid only
    after `import`; qualified names in types, calls and patterns.

Author's decision (2026-09-25):

14. **Builds obtain pinned content** ([DEP-9]): a build may download content
    the lock pins and that is missing locally — only the locked revision,
    verified against the locked content hash before use, reported per
    module. It never selects versions, requests anything the lock does not
    pin, writes the manifest or the lock, or falls back to another source.
    `--offline` forbids every request.

## Alternatives

- **Names from a central registry** (as crates.io or npm: `import
  "collections"` resolved by one index). Rejected: it contradicts items 1
  and 2, makes one service the naming authority and a dependency of every
  user, and invites name squatting. Its advantages — short names, one place
  for policy — can come later from an optional catalog that never decides
  resolution.
- **Bare names mapped by the manifest** (a local name in the source, the
  origin in the manifest, as in Cargo dependency keys or import maps).
  Rejected: the source alone no longer says which code it uses, a reader
  needs the manifest, and two projects can give one name to different code.
- **Go-compatible `go.mod`, `go.sum` and module proxy protocol.** Rejected:
  Yxy is not Go (decision 0001); it would bind Yxy's formats to Go's
  evolution and make Go tooling semantics implicit. The ideas are borrowed
  (origin paths, `/vN`, replacements only in the main module, minimal version
  selection); the formats and protocol are Yxy's own.
- **Package = file** (today's [PRG-1]). Deferred (OPEN #15): lowest context
  per unit, but paths would name files, and splitting a library into files
  would force helpers to become public.
- **Keep `module <name>` as the package clause.** Rejected for now
  (OPEN #16): the word would name both the versioned unit and the package.
- **Version ranges with a constraint solver** (Cargo, npm). Rejected for
  now: a new release can change the result unless a lock intervenes; the
  solver is complex; minimal version selection gives item 3 by construction.
- **Builds that never use the network**, with an explicit `yxy fetch` before
  the first build. Rejected by the author (item 14): one more step after
  every clone; `--offline` and `yxy fetch` keep that mode available.

## Consequences

- The compiler must load several files per package, resolve qualified
  names, check visibility, detect import cycles and derive symbols per
  package — all before any network code exists.
- `package` becomes reserved now (a small change to the lexer); the
  compiler's corpus is rewritten when packages are implemented.
- Adding an effect to a `pub fn` is a breaking change of a module.
- The dependency tooling must be testable offline, with local repositories
  as origins; the package service is optional and cannot be a prerequisite.
- The local store and its verification follow implementation decision 0004
  (CACHE-01 to CACHE-03).

## What could change it

The author's review of any experimental item; a reading or agent study
comparing directory and file packages; real origins that need uppercase
paths, untagged revisions, target-specific files or native code;
compatibility breaks under minimal version selection; the design of the
service protocol and of a trust model (checksum log or signatures).
