# Contribution README — binaryen issue #3244

- **Project:** [WebAssembly/binaryen](https://github.com/WebAssembly/binaryen) — compiler/toolchain infrastructure library for WebAssembly (C++)
- **Issue:** [#3244 — Optimize trampled data in MemoryPacking](https://github.com/WebAssembly/binaryen/issues/3244) (labels: `good first bug`, `help wanted`)
- **Background PR:** [#3222 — Fuzz fix for MemoryPacking on trampled data](https://github.com/WebAssembly/binaryen/pull/3222)
- **My fork:** https://github.com/JPL11/binaryen
- **Working branch:** https://github.com/JPL11/binaryen/tree/fix-issue-3244

## Reproduction Process

### Environment Setup

Binaryen is a C++17 project built with CMake. There is no dev container; the README's
build instructions are accurate and minimal.

Setup on macOS (Apple Silicon, macOS 26.4):

1. Installed build tools: `brew install cmake ninja` (Xcode CLT clang was already present).
2. Cloned my fork: `git clone https://github.com/JPL11/binaryen.git`
3. Added upstream and confirmed my fork's `main` is current:
   `git remote add upstream https://github.com/WebAssembly/binaryen.git && git fetch upstream main`
4. Initialized submodules (googletest, mimalloc): `git submodule update --init`
5. Configured and built just the tool I need (much faster than building everything):
   ```sh
   cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
   ninja -C build wasm-opt
   ```

**Challenges:** none significant — `cmake`/`ninja` were not installed initially
("cmake not found"), fixed via Homebrew. Full `wasm-opt` build took only a few
minutes with ninja (279 targets).

### Steps to Reproduce

The issue is a missing optimization, not a crash: when active data segments
*overlap* ("trample" each other), the `memory-packing` pass detects the overlap
and gives up entirely instead of optimizing.

1. Create `repro/trampled.wast` — two active segments where the second writes a
   zero over the `"x"` written by the first:
   ```wat
   (module
    (memory $0 1 1)
    (data (i32.const 1024) "x")
    (data (i32.const 1024) "\00")
   )
   ```
2. Run: `./build/bin/wasm-opt repro/trampled.wast --memory-packing -S -o -`
3. **Expected (per issue #3244):** since the `"x"` is immediately overwritten by
   `0` and memory is zero-initialized, the final memory image is all zeros — both
   segments could be dropped entirely.
4. **Actual:** the pass prints
   `warning: active memory segments have overlap, which prevents some optimizations.`
   and emits both segments completely unchanged.
5. Control case (`repro/control.wast`, identical but second segment at offset
   1025 so there is no overlap): the pass works as designed and drops the
   all-zero segment, leaving only `(data $0 (i32.const 1024) "x")`.

Reproduced twice with identical results (the tool is deterministic). The
repository's own lit test encodes today's give-up behavior at
`test/lit/passes/memory-packing_all-features.wast:2196-2230` ("when we see one
bad thing, we give up") — those CHECK expectations will change with this fix.

## Solution Approach

### Understand

At instantiation, active data segments are written to memory in module order, so
a later segment can overwrite ("trample") bytes written by an earlier one.
PR #3222 made `MemoryPacking` *safe* in this case: if any two active segments
overlap, the pass refuses to optimize the module's segments at all, because
optimizing out a zero in a later segment would be wrong (that zero may be needed
to overwrite earlier data). Issue #3244 asks for the follow-up: instead of
giving up, *compute the outcome of the trampling statically* and never emit the
trampled bytes in the first place.

### Match — root cause location

- `src/passes/MemoryPacking.cpp`, `MemoryPacking::canOptimize()`:
  - Line 250: `// TODO: optimize in the trampling case` (left by the maintainer in #3222)
  - Lines 251–263: a `DisjointSpans` check (from `src/support/space.h`) returns
    `false` — disabling the whole pass — as soon as any two active
    constant-offset segments overlap.
- Importantly, by the time this check runs, **all active segments are known to
  have constant offsets** (non-constant offsets already returned `false` at line
  239–241). So the final memory image is fully computable at compile time.
- The pass already has all the machinery to exploit this: `calculateRanges()`
  splits segments around zero ranges and drops them. If trampled bytes in
  earlier segments are removed *before* packing, the existing machinery does the
  rest.

### Plan

1. Add a normalization step (e.g. `MemoryPacking::removeTrampling()`), run from
   `run()` before `canOptimize()`. Algorithm — iterate active constant-offset
   segments in **reverse** order, maintaining an interval set of bytes covered
   by later segments:
   - For each segment, subtract the covered intervals from its span and split it
     into one segment per surviving sub-interval (slicing its data and adjusting
     the constant offset). Bytes covered by a later segment are dropped — they
     are guaranteed to be overwritten before any code can observe them.
   - Add the segment's original span to the covered set and continue.
   - After this, all active segments are disjoint, the existing `DisjointSpans`
     check in `canOptimize()` passes naturally, and normal packing proceeds —
     no changes needed to the packing logic itself.
2. Conservative gating (correctness caveats to confirm with maintainers on the issue):
   - **Trap observability:** with bulk memory, segments initialize in order and a
     later out-of-bounds segment traps mid-instantiation. If the memory is
     *imported*, earlier segments' partial writes are observable after a failed
     instantiation, so dropping trampled bytes could be visible. Gate: only
     apply when the memory is not imported, or when every segment is provably
     in-bounds (`offset + size <= initial pages * 64KiB`), or under
     `trapsNeverHappen`.
   - **Referrers:** skip segments referenced by `data.drop`/GC instructions
     (mirror the existing `canSplit()` checks), since splitting changes indices.
3. Files to touch:
   - `src/passes/MemoryPacking.cpp` — new normalization step + remove/relax the give-up path
   - possibly `src/support/space.h` — interval-subtraction helper on `DisjointSpans`
   - `test/lit/passes/memory-packing_all-features.wast` — update the four
     existing trampling modules (lines ~2196–2230) whose CHECKs encode the old
     give-up behavior, and add new cases: full overlap (both dropped), partial
     overlap, non-zero trampling non-zero (`"x"` overwritten by `"y"` → only
     `"y"` survives), multiple tramplers, imported-memory gating.

### Implement

*Phase III — placeholder.* Branch: https://github.com/JPL11/binaryen/tree/fix-issue-3244

### Review

- Binaryen's `CONTRIBUTING.md` defers to the WebAssembly design repo's
  contributing guidelines; tests live in `test/lit/` and CHECK lines are
  regenerated with `scripts/update_lit_checks.py`.
- Recent commit messages follow `Short imperative summary (#PR)` style; PRs are
  squash-merged, so a clear PR title/description matters most.
- I have commented on the issue to claim it; before implementing I will post
  this plan (especially the trap-observability gating question) on the issue
  for maintainer feedback, since @kripken left the TODO and triaged the issue.

### Evaluate

- New/updated lit tests above must pass: `build/bin/binaryen-lit test/lit/passes/memory-packing_all-features.wast`
  (or `python3 check.py lit` for the whole lit suite).
- Full test suite (`python3 check.py`) to confirm no regressions.
- Manual check: the reproduction case must now emit a module with **no data
  segments** and no warning; the control case must be unchanged.

---
**Phase II Complete.**
