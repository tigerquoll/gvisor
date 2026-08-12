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
| 2 | Pointer-typed package-variable guards resolve inconsistently | silent non-enforcement + false positives | Candidate issue, evidence complete |
| 3 | Annotations on bare package-level vars are silent no-ops | silent non-enforcement | Candidate issue, evidence complete |
| 4 | Undocumented behaviors relied on in practice | doc gap | Partially addressed in #14078 |

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

## 2. Pointer-typed package-variable guards resolve inconsistently (CANDIDATE ISSUE)

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

Suggested thesis for the issue: annotations naming a field of a pointer-typed
package-level variable should resolve through the pointer (matching acquisitions),
or be rejected at parse time — never silently mismatch.

## 3. Annotations on bare package-level vars are silent no-ops (CANDIDATE ISSUE)

`+checklocks:<globalMu>` attached to a bare package-level variable parses without
complaint and enforces nothing. Field annotations bind to struct fields only.
Verified in one run: a guarded field of a global struct is enforced
(`invalid field access ... {global:...}`), while a bare var carrying the identical
annotation produces no diagnostic for an unguarded read.

This is a green-washing hazard: the annotation looks like coverage and is coverage
of nothing. The README documents what may serve as a *guard* but never states that
the guarded *entity* must be a struct field. Suggested fix: warn (or error) at parse
time when a lock annotation is attached to a non-field entity; document the
restriction either way.

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
- Finding 2 probes: four probe variants (unguarded read / guarded read /
  exclusion-while-held direct and via accessor) plus the dual-resolution
  precondition probe, run against a pointer-typed global; reproducible on any such
  variable.
- Finding 3 probe: identical annotation on a bare var vs a global struct field,
  one run, one diagnostic.
