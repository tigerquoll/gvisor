# checklocks tool findings — August 2026

Findings about `tools/checklocks` made while adopting it across two production Go
codebases (Apache YuniKorn core and k8shim: ~35 packages, several hundred annotations
across every annotation class). One bug fixed and filed upstream; two candidate
issues with complete evidence; one documentation gap partially addressed. All
reproduction claims were verified empirically, most in both the pre-fix and post-fix
tool builds.

| # | Finding | Severity | Status |
|---|---|---|---|
| 1 | Panic on cross-package use of unexported global guards | crash | **Fixed — google/gvisor#14078** (CLA green, in review) |
| 2 | Pointer-typed package-variable guards resolve inconsistently | silent non-enforcement + false positives | **Fixed — fork PR #3** (upstream submission pending) |
| 3 | Guard annotations silently dropped on some `var` declaration forms (corrected diagnosis; originally "bare vars can't be guarded") | silent non-enforcement | **Fixed — fork PR #4** (upstream submission pending) |
| 4 | Undocumented behaviors relied on in practice | doc gap | Partially addressed in #14078 / PR #4 |

## 1. Panic: unexported global guard, cross-package use (FIXED)

Any annotation whose guard is an unexported package-level variable panicked the
analyzer when the annotated function was called, or the annotated field accessed,
from another package:

```
panic: interface conversion: interface is nil, not ssa.Value
    (*globalGuard).resolveCommon   tools/checklocks/facts.go:208
```

Both resolution paths were affected — `resolveCall` (function annotations) and
`resolveField` (field annotations). Verified matrix, one two-package fixture:

| guard | call path | field path |
|---|---|---|
| unexported global | panic | panic |
| exported global | works, enforces | works, enforces |

Root cause: `pkg.Members[g.ObjectName].(ssa.Value)` unchecked; export data omits
unexported package-level variables, so cross-package the member lookup is nil.
History: #7721 (2022) fixed the exported cross-package case only; the field-path
variant has been latent since, and the call path became commonly reachable when
`checklocksexclude{,write}` landed (#12439, Jan 2026).

Fix (in #14078): an explicit `resolvedValue.unavailable` state, skipped silently at
five use sites. Silent skip is sound — a consumer could only hold the unresolvable
guard via an acquire-annotated API whose fact references the same unresolvable
object, so the acquisition could never enter its lock state either; the skipped
check is vacuously unviolatable. A plain "invalid" return would instead have emitted
`cannot be resolved` false positives in some paths and merely relocated the panic
into `lockState.isHeld`/`lockField` in others. The README now documents the
limitation (unexported global locks are enforceable only within their declaring
package).

Validation beyond the in-repo tests: fixed-vs-pinned binaries produced
**byte-identical output over both fully annotated YuniKorn repositories** — clean
trees, and with injected diagnostics covering every annotation class — while the
pre-fix binary panicked on exactly the packages that are cross-package users of
newly added global-guard annotations.

## 2. Pointer-typed package-variable guards resolve inconsistently (FIXED — fork PR #3)

For `var d = &dispatcher{...}` (a pointer-typed package-level variable), a FUNCTION
annotation naming `d.lock` resolves to the field of the variable itself, while an
acquisition or FIELD guard resolves through the pointer. Captured side by side from
one probe run:

```
must hold d.lock ... (&({global:d}.lock))     to call probeNeedsLock,
    but not held        (locks: &(*({global:d}).lock) exclusively)     <- lock IS held
invalid field access, lock (&(*({global:d}).lock)) must be locked ...
                        (locks: &({global:d}.lock) exclusively)        <- annotation-held
```

The two forms never unify. Consequences, all verified:
- `+checklocksexclude:d.lock` on a self-locking API is **silently inert** — a caller
  holding the lock and invoking the excluded function is not reported, even
  same-package.
- `+checklocks:d.lock` preconditions **false-positive on correctly locked callers**
  and inside the annotated body.
- No spelling escapes it: `(*d).lock` and `*d.lock` are rejected at annotation parse.
- Field guards and plain acquisition tracking through the pointer work fine — only
  the function-annotation path mismatches.
- Identical behavior in pre- and post-#14078 builds: pre-existing and independent.

Root cause (established by the fix): the `ssa.Value` for a package-level variable is
the ADDRESS of the variable, one indirection above its declared type. A value-typed
global needs only `FieldAddr`; a pointer-typed global needs a load first, and
`maybeFindFieldListObj`/`resolveStruct` silently chased the pointer while computing
field indices, discarding the fact that a dereference was required. The same defect
covered two further variants the probes had not reached: a package variable that IS
a pointer-typed lock (`var muPtr = &sync.Mutex{}`), and interface-typed variables.

Fix (fork PR #3): `globalGuard` records whether resolution must dereference the base
value; applied at resolution so annotations unify with acquisitions. Before/after:
inert exclusions now report, false-positive preconditions now clean, value-typed
controls unchanged, whole-test-tree otherwise identical. Externally validated on the
YuniKorn shim dispatcher probes (the motivating case). Two honest limits recorded in
the PR: a lock reached via an accessor function (`d := getDispatcher()`) still does
not unify (value-flow aliasing, a separate pre-existing limitation), and #14078's
unexported-guard skip still applies cross-package.

## 3. Guard annotations dropped on some `var` declaration forms (FIXED — fork PR #4; corrected diagnosis)

Original (wrong) diagnosis: "bare package-level vars cannot be guarded entities."
That is refuted by gVisor's own `test/crosspkg`, which guards the package-level
`Foo` with `+checklocks:FooMu` and asserts enforcement — passing on master.

Actual defect: COMMENT PLACEMENT. For a parenthesized `var (...)` block the doc
comment attaches to the inner `ValueSpec`; for a single non-parenthesized `var` it
attaches to the `GenDecl`, and the fact extractor read only `ValueSpec.Doc` — so
`GenDecl.Doc`, trailing `ValueSpec.Comment`, and second-and-later names in a spec
were all silently dropped. (Types already handled all three comment positions;
globals were missing the same treatment.) Our probe happened to use a dropped
placement, hence the over-general first diagnosis.

Fix (fork PR #4): read all three comment positions and apply to every declared name,
mirroring the existing type-alias handling. A briefed alternative — rejecting
annotations on non-field entities — would have BROKEN working code and was correctly
not implemented. gVisor-tree impact: zero (no annotations in affected placements;
verified by scan and by before/after runs over the seven most-annotated packages).
README now states where global-variable guard annotations may be placed.

Still a green-washing lesson for adopters: an annotation in a dropped placement was
silently inert. The canary-style self-test pattern (assert a known violation is
detected) is what catches this class in CI.

## 4. Documented-behavior gaps relied on in practice

- Trailing line-level `// +checklocksignore` works (keyed by file:line via
  `extractLineFailures`) and is load-bearing in real adoption (closures cannot carry
  annotations and do not inherit a containing function's ignore), but the README
  documents `+checklocksignore` only for functions and fields. Worth documenting —
  or the behavior may be narrowed someday and silently strand adopters.
- The unexported-global limitation is now documented (#14078).
- The `-inferred` suggestion pass is observation-ratio based and unstable run to
  run as ignores/annotations are added nearby; adopters gating CI on the analyzer
  will likely want `-inferred=false` plus intentional annotation. A README note on
  that trade-off would help.

## Reproduction assets

- #14078 branch: `checklocks-global-guard-fix` (this fork) — fix, tests
  (`test/crosspkg` + `test/globals.go`, `+checklocksfail` expectations verified
  load-bearing), README notes.
- Two-commit review copy: fork PR #1 (`checklocks-unexported-global-guard`).
- Finding 2 fix: fork PR #3 (`checklocks-pointer-global-guards`) — tests in
  test/globals.go + crosspkg cover pointer-typed struct globals, pointer-typed lock
  variables, and interface-typed variables; externally validated on the YuniKorn
  shim dispatcher probes.
- Finding 3 fix: fork PR #4 (`checklocks-annotation-placement-check` branch name
  retained from the original diagnosis) — tests cover all three previously-dropped
  comment placements.
- Merge-order note: #14078, PR #3 and PR #4 all touch test/globals.go (PR #3 also
  overlaps #14078 in facts.go/crosspkg); they have not been tested merged together.
