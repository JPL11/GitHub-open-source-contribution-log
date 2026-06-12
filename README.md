# Contribution 1: Optimize trampled data in MemoryPacking

**Contribution Number:** 1
**Student:** Jacky Li (GitHub: [JPL11](https://github.com/JPL11))
**Issue:** https://github.com/WebAssembly/binaryen/issues/3244
**Status:** Phase III Complete

---

## Why I Chose This Issue

Binaryen is the optimizer/toolchain library behind Emscripten and many other
WebAssembly compilers, so improvements here affect real-world binary sizes for
a lot of users. This issue is labeled `good first bug` + `help wanted`, was
triaged by the lead maintainer (@kripken), and has a crisp, self-contained
scope: a single optimization pass (`MemoryPacking`), with the relevant
background already written up in PR #3222. It also let me learn how a
production compiler pass reasons about *observable semantics* (instantiation
order, traps, imported memory) rather than just shuffling bytes — exactly the
kind of systems thinking I wanted practice with.

---

## Understanding the Issue

### Problem Description

WebAssembly modules initialize linear memory with *data segments*. Active
segments are applied in order at instantiation, so a later segment can
overwrite ("trample") bytes written by an earlier one — this is legal wasm.
PR #3222 made the `memory-packing` pass *safe* in this situation by detecting
any overlap between active segments and refusing to optimize the module's
segments at all. Issue #3244 asks for the proper fix: since only the final
memory contents are observable, trampled data should simply never be emitted.

### Expected Behavior

`wasm-opt --memory-packing` should optimize modules with overlapping active
segments, dropping bytes that are guaranteed to be overwritten. E.g. for
`(data (i32.const 1024) "x")` followed by `(data (i32.const 1024) "\00")`,
final memory is all zeros, so *both* segments should be removed.

### Current Behavior

The pass prints `warning: active memory segments have overlap, which prevents
some optimizations.` and leaves every segment in the module untouched.

### Affected Components

- `src/passes/MemoryPacking.cpp` — the `MemoryPacking` pass (`canOptimize()`)
- `src/support/space.h` — `DisjointSpans`, the overlap detector added in #3222
- `test/lit/passes/memory-packing_*.wast` — lit tests for the pass

---

## Reproduction Process

### Environment Setup

Binaryen is a C++17 / CMake project. On macOS (Apple Silicon):

- `brew install cmake ninja` (clang from Xcode CLT was already present)
- `git submodule update --init` (googletest, mimalloc)
- `cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release && ninja -C build`
- Test tooling: `python3.12 -m venv .venv && .venv/bin/pip install -r
  requirements-dev.txt` (`lit`, `filecheck`, `ruff`; `.venv/` is already in the
  project's `.gitignore`). Note: `check.py` requires Python ≥ 3.10 — the macOS
  system Python (3.9) fails an assert, which cost me a few minutes.
- `brew install clang-format` for the project's enforced C++ style.

### Steps to Reproduce

1. Save as `repro/trampled.wast`:
   ```wat
   (module
    (memory $0 1 1)
    (data (i32.const 1024) "x")
    (data (i32.const 1024) "\00")
   )
   ```
2. Run `./build/bin/wasm-opt repro/trampled.wast --memory-packing -S -o -`
3. **Observed (before fix):** warning about overlapping segments; both segments
   emitted unchanged. **Expected:** both segments removed (final memory is all
   zeros). A control file with the second segment at 1025 (no overlap) shows
   the pass optimizing normally.

### Reproduction Evidence

- **Branch:** https://github.com/JPL11/binaryen/tree/fix-issue-3244
- **My findings:** the bail-out is in `MemoryPacking::canOptimize()`
  (`src/passes/MemoryPacking.cpp`), where the maintainer left
  `// TODO: optimize in the trampling case`. The repo's own lit tests encoded
  the give-up behavior ("when we see one bad thing, we give up").

---

## Solution Approach

### Analysis

By the time the overlap check runs, `canOptimize()` has already verified that
*every* active segment has a constant offset (non-constant offsets bail out
earlier). That means the final memory image is fully computable at compile
time — the conservative bail-out is throwing away information it already has.

### Proposed Solution

Zero out every byte that a later segment overwrites, then let the pass's
existing zero-dropping machinery remove those bytes. This is the minimal
change: segment count, offsets, and order are untouched, so there is no
interaction with segment referrers (`memory.init`/`data.drop`), segment
indices, or trap timing. Because emitted segments preserve their original
order, any residual overlap (a small zero gap kept merged inside an earlier
segment) is still overwritten correctly by the later segment.

One safety gate: if the memory is **imported**, a later out-of-bounds segment
traps mid-instantiation and the partially-written state stays visible in the
importing module — so there we keep the old behavior (bail + warning).
For module-defined memory, a failed instantiation can never expose the memory,
so the transformation is unobservable.

### Implementation Plan

**Understand:** Active segments apply in order before any code runs; only
final memory contents are observable; trampled bytes are dead data.

**Match:** The pass already drops zero bytes from active segments (memory is
zero-initialized) — trampled bytes can be turned into exactly that case.
`DisjointSpans` (`src/support/space.h`) already detects the overlap.

**Plan:**
1. In `canOptimize()`, on detected overlap: bail only if the memory is
   imported; otherwise call a new `zeroOutTrampledData()`.
2. `zeroOutTrampledData()`: iterate segments in reverse, maintaining a map of
   disjoint covered regions; zero each segment's bytes that later segments
   cover; merge the segment's own span into the map. O(n log n).
3. Update the lit tests that asserted the give-up behavior; add new cases.

**Implement:** branch `fix-issue-3244`, commits:
- `3b1e2dc0d` — MemoryPacking: optimize trampled data instead of giving up
- `a90c6fe9a` — Add and update tests for trampled data optimization

**Review:** clang-format clean; diff scoped to 3 files; commit messages match
the project's imperative style (PRs are squash-merged); CHECK lines generated
with `scripts/update_lit_checks.py --all-items` per the file headers.

**Evaluate:** see Testing Strategy below.

---

## Testing Strategy

### Unit Tests

Lit tests (the project's standard for pass changes), regenerated with
`scripts/update_lit_checks.py`:

- [x] Full trample by zero → both segments removed
  (updated existing test in `memory-packing_all-features.wast`)
- [x] Trampling in one segment doesn't block optimizing others (updated
  existing "we give up" test)
- [x] Full trample by non-zero byte → only the later byte remains
- [x] Partial trample in the middle of a segment (`"xyz"` + `"A"` at 1025)
- [x] Chained trampling where tramplers are themselves trampled
  (`"abc"`/`"de"`/`"f"` all at 1024 → `c`,`e`,`f` segments)
- [x] One segment trampling multiple earlier ones (`"WXYZ"` over `"ab"`+`"cd"`)
- [x] Passive segments are neither trampling nor trampled
- [x] Imported memory + trampling → not optimized (gate), in
  `memory-packing_zero-filled-memory.wast`

### Integration Tests

- [x] Full lit suite: 928 passed, 0 failed (`build/bin/binaryen-lit test/lit`)
- [x] C++ unit tests: 350/350 passed (`build/bin/binaryen-unittests`)
- [x] `check.py` full suite from the venv

### Manual Testing

- `wasm-opt --fuzz-exec --memory-packing` on a hand-written module reading
  every byte involved in trampling: identical results before/after the pass.
- Randomized check (`repro/fuzz_trampling.py`, kept out of the PR): 300 random
  modules with 2–6 overlapping segments (random offsets/lengths/zero-heavy
  data), each executed before and after the pass — 300/300 identical memory
  contents.

---

## Implementation Notes

### Week 1 Progress (June 11)

**What I built:**
- `zeroOutTrampledData()` in `src/passes/MemoryPacking.cpp` (+72 lines incl.
  comments) and the imported-memory gate in `canOptimize()`; removed the
  `TODO: optimize in the trampling case`.
- 6 new lit test modules + updated 3 existing ones across two test files.

**Challenges faced / decisions made:**
- My Phase II plan was to *split* trampled segments into sub-segments. While
  re-reading the pass I realized splitting interacts with segment referrers,
  indices, and trap semantics. Zeroing trampled bytes instead reuses the
  pass's existing zero-dropping machinery and changes nothing structural —
  a much smaller, safer diff. Plans meeting reality, as advertised.
- The subtlest question was *when the transformation is observable*: I had to
  reason through wasm instantiation semantics (segments apply in order, before
  any code; a trapping segment aborts instantiation but earlier writes stand).
  That's where the imported-memory gate comes from.
- Test tooling friction: `filecheck`/`lit` come from `requirements-dev.txt`,
  and `check.py` needs Python ≥ 3.10 (system Python is 3.9) — solved with a
  project venv.

### Code Changes

- **Files modified:** `src/passes/MemoryPacking.cpp`,
  `test/lit/passes/memory-packing_all-features.wast`,
  `test/lit/passes/memory-packing_zero-filled-memory.wast`
- **Key commits:**
  [`3b1e2dc0d`](https://github.com/JPL11/binaryen/commit/3b1e2dc0d) (fix),
  [`a90c6fe9a`](https://github.com/JPL11/binaryen/commit/a90c6fe9a) (tests)
- **Approach decisions:** zero-out instead of split (see above); conservative
  gate for imported memory with a possible follow-up to allow it when all
  segments are provably in-bounds (`maxAddress <= initial pages * 64KiB`).

---

## Pull Request

**PR Link:** _to be opened in Phase IV_

**PR Description (draft):**

> ### MemoryPacking: optimize trampled data instead of giving up (#3244)
>
> When active segments overlap, a later segment overwrites ("tramples") the
> data of an earlier one. #3222 made the pass give up on any overlap. But
> since active segments are applied in order during instantiation, before any
> code can run, only the final contents of memory are observable — so we can
> zero out all trampled bytes and let the normal optimization of zeros remove
> them. This leaves segment count, offsets, and order untouched, so referrers
> and trap behavior are unaffected.
>
> We keep the old behavior (warn + skip) when the memory is imported: there, a
> later out-of-bounds segment traps mid-instantiation and the partially
> written state remains visible in the importing module, so even trampled
> data matters. (Possible follow-up: optimize imported memories too when all
> segments are provably within the declared minimum size.)
>
> Verified with updated/new lit tests, plus fuzz-exec comparison on 300 random
> overlapping-segment modules (identical memory contents before/after).
>
> Fixes #3244

**Maintainer Feedback:**
- _pending — will post the approach on the issue when opening the PR_

**Status:** Phase III complete, PR not yet opened

---

## Learnings & Reflections

### Technical Skills Gained

- How wasm linear-memory instantiation actually works: active vs passive
  segments, application order, bulk-memory trap semantics, why imported
  memory outlives a failed instantiation.
- Compiler-pass hygiene: making the *minimal* semantically-justified
  transformation and letting existing machinery (zero-splitting) do the rest.
- Binaryen's test workflow: lit + FileCheck with auto-generated assertions,
  `--fuzz-exec` as a built-in differential testing tool.

### Challenges Overcome

- Revised my own Phase II plan when the splitting approach turned out to have
  hidden interactions; the zeroing approach came from asking "what does the
  existing pass already know how to delete?"
- Reasoned about observability edge cases (imported memory + mid-instantiation
  trap) instead of assuming the happy path.

### What I'd Do Differently Next Time

- Set up the project venv and `requirements-dev.txt` *before* touching tests.
- Ask the maintainer to confirm the imported-memory gating earlier — I'll
  highlight it as the main review question in the PR.

---

## Resources Used

- https://github.com/WebAssembly/binaryen/pull/3222 (background fix + design notes)
- https://webassembly.github.io/spec/core/exec/modules.html (instantiation order & trap semantics)
- https://github.com/WebAssembly/bulk-memory-operations/blob/master/proposals/bulk-memory-operations/Overview.md (segment init semantics)
- `scripts/update_lit_checks.py` and existing tests in `test/lit/passes/` as patterns
