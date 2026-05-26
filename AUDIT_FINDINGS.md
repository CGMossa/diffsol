# diffsol audit — `audit/sweep` (off `main`)

Independent sweep for "weird things" across `diffsol/`, `diffsol-c/`, `examples/`,
`book/`, and the test suites. Findings only — no fixes were made. Severity is the
auditor's judgement; near-duplicates have been collapsed. False positives surfaced
by the per-area agents have been pruned. Findings are spot-checked against the
current source, but a handful (marked *unverified*) are reported as-flagged.

The doc work on `doc/limited-vs-granular-api` is included in scope: the diff
against `main` is purely additive rustdoc plus one SUMMARY entry plus a one-line
typo fix in `adjoint_sens.md`; nothing weird in the change itself.

---

## Highest priority (read these first)

1. **`examples/physics-based-battery-simulation/Cargo.toml:2`** — package `name = "physics-based-batter-simulation"`. Directory name has the `y`, the package name does not. *(medium → high if anything depends on the package name string downstream.)*
2. **Three empty stub pages in the book:**
   - `book/src/solve/stopping_time.md` (2 lines)
   - `book/src/solve/stopping_event.md` (1 line)
   - `book/src/primer/spatial_population_dynamics.md` (1 line)
   None of them are linked from `SUMMARY.md`, so they're orphan stubs — not crashing the build, but signal abandoned chapters.
3. **`book/src/solve/stopping.md`** is in the TOC and exists, but `stopping_time.md` and `stopping_event.md` look like they were meant to be subpages of it and were never written.
4. **`diffsol-c/src/host_array.rs:265–270`** and **`:280`** — `unsafe { ArrayView2::from_shape_ptr(...) }` and `unsafe { std::slice::from_raw_parts(self.ptr as *const T, len) }` build views/slices from `self.ptr` without checking it for null. If a `HostArray` ever holds a null `ptr` with non-zero `len/shape`, that's UB.
5. **`diffsol-c/src/ode_solver_type.rs:263, 268, 389`** — three live TODO/"for now" comments around adjoint checkpoint handling: "can we avoid cloning here?", "we will only consider a single output g for now, so nout_override is 1", "TODO: remove clone here". The single-output-only constraint in particular is a real semantic limit on adjoint behaviour that isn't surfaced in user-facing docs.

---

## diffsol/ (core crate)

### Suspicious panics / unwraps on user-reachable paths

- **[medium]** `diffsol/src/jacobian/coloring.rs:13` — `.unwrap()` on `non_zeros.iter().max()` is guarded by an emptiness check above but the unwrap-after-guard pattern is fragile to refactors.
- **[medium]** `diffsol/src/nonlinear_solver/root.rs:89` and `:134` — `IndexType::try_from(imax).unwrap()` / `IndexType::try_from(imax_i32).unwrap()` will panic if `imax` overflows `IndexType` rather than returning a proper error.
- **[medium]** `diffsol/src/linear_solver/cuda/lu.rs:75–77, 117–118, 159–161` — multiple `i32::try_from(dim).unwrap()` on matrix dimensions; unlikely to fire but undefended.
- **[medium]** `diffsol/src/matrix/cuda.rs:102, 122, 145, 188, 196, …` — heavy `.expect()` on CUDA driver/runtime calls. Production paths panic on GPU OOM, kernel launch failure, etc. With the `cuda` feature off this is dormant; with it on, this is the entire error mode.
- **[medium]** `diffsol/src/context/cuda.rs:109–113, 120` — `Self::new(0).unwrap()` in `Default` impl and a panic in `function()` on kernel-not-found.

### `unimplemented!()` in trait surfaces

- **[medium]** `diffsol/src/ode_equations/mod.rs:309, 313` — `set_params()` / `set_model_index()` on the base `OdeEquations` trait default to `unimplemented!()`. Default-impl-as-panic for "expected to be overridden" methods means downstream implementors who forget to override get a runtime crash with no compile-time signal. Consider documenting the requirement explicitly on the trait (or making them non-defaulted).
- **[medium]** `diffsol/src/ode_equations/mod.rs:92–184` — the `NoAug<Eqn>` wrapper implements ~30 trait methods as `panic!("This should never be called")`. Type-system trick for an unreachable state; correct today, but no safety doc explains the invariant.
- **[low]** `diffsol/src/ode_equations/test_models/foodweb.rs:840, 843` and `heat2d.rs:213` — `unimplemented!()` in test-model helpers. Fine for tests; flagged for completeness.

### Errors that get dropped

- **[medium]** `diffsol/src/linear_solver/suitesparse/klu.rs:199` — `.ok()` silently discards the error from `KluNumeric::try_from_symbolic()`. Downstream `solve()` will then fail with a less informative "numeric not initialized" rather than the real factorization error.
- **[medium]** `diffsol/src/linear_solver/suitesparse/klu.rs:229` — same pattern on `KluSymbolic::try_from_matrix()`.
- **[medium]** `diffsol/src/ode_equations/adjoint_equations.rs:691` — `Err(_) => panic!(...)` discards the inner error before panicking.

### Dead / commented-out code

- **[low]** `diffsol/src/linear_solver/cuda/lu.rs:180–290` — ~108 lines of commented-out C/cusolver pseudocode (a verbatim NVIDIA `cusolverDnDgetrf` example transcribed with `//` prefixes). Reference material left in source.
- **[low]** `diffsol/src/ode_solver/adjoint.rs:551–557` — commented-out loop body.
- **[low]** `diffsol/src/ode_solver/sdirk.rs:472, :479` — commented-out `let linear_solver = …` / `let error_norm = …`.
- **[low]** `diffsol/src/jacobian/mod.rs:96` — commented-out `// if v.is_nan()` line.
- **[low]** `diffsol/src/ode_equations/test_models/heat2d.rs:315` and `exponential_decay_with_algebraic.rs:179` — commented-out helper/test setup.

### Stale TODOs / design debt

- **[medium]** `diffsol/src/jacobian/mod.rs:18–20, 78–79` — two near-identical "TODO: not efficient for non-host vectors, ok for now" notes around sparsity detection. Real performance debt.
- **[low]** `diffsol/src/linear_solver/suitesparse/klu.rs:109` — "TODO: there is also `klu_refactor` which is faster and reuses inner structure". Optimisation never landed.
- **[low]** `diffsol/src/ode_equations/adjoint_equations.rs:123` — "todo: this seems a bit hacky, perhaps a dedicated function on the trait for this?" — DiffSL-specific workaround.
- **[low]** `diffsol/src/ode_equations/diffsl.rs:1185` — "todo: would rhs_srgrad ever use rr? I don't think so, but need to check" — unresolved uncertainty.
- **[low]** `diffsol/src/vector/mod.rs:233, :240` — "TODO: would prefer to use `From` trait but not implemented for `faer::Col`".

### Typos / minor

- **[low]** `diffsol/src/op/mod.rs:32` — doc string: `paramter` → `parameter`.
- **[low]** `diffsol/src/ode_solver/bdf.rs:1288` — comment: `reinitalise` → `reinitialise`.
- **[low]** `diffsol/src/ode_solver/runge_kutta.rs:438` — comment: `reinitalise` → `reinitialise`.
- **[low]** `diffsol/src/ode_solver/mod.rs:970` — comment "do three more steps after the 1st bound and many sure they are correct" — `many` should be `make`.

---

## diffsol-c/ (FFI + runtime dispatch)

### Unsafe blocks

- **[high]** `diffsol-c/src/host_array.rs:265–270, :280` — see "Highest priority" above. Null check missing on `self.ptr` before the slice / `ArrayView2::from_shape_ptr` build.
- **[medium]** `diffsol-c/src/ode_c.rs:85–97` — `unsafe fn dependency_pairs_from_raw_parts` does null-check `deps_ptr` but has no `// SAFETY:` comment for the `from_raw_parts` invocation.
- **[medium]** `diffsol-c/src/ode.rs:31–32` — `unsafe impl Send for Ode {}` and `unsafe impl Sync for Ode {}` with no safety comment. Likely correct (the wrapper holds `Arc<Mutex<…>>`), but unsafe trait impls should justify themselves at the site.

### Panics on FFI paths

- **[high]** `diffsol-c/src/solve.rs:470` (and dozens more at `:600, 612, 620, 628, 644, 655, 669, 683, 695, 706, 752, 759, 775, 779, 786, 802, 827, 850, 873`) — `M::T::from_f64(x).unwrap()` inside Solve impls reachable from the C FFI. A panic crossing the C boundary is UB. Most fire only for out-of-range f64→scalar conversions (e.g., when the scalar type is f32 or a fixed-point type), so in practice this is dormant for the default f64 build but live for non-f64 backends.
- **[medium]** `diffsol-c/src/solve_macros.rs:31, 41` — the `option_value_to_store!` / `option_value_from_store!` macros expand the same `.unwrap()` everywhere they're used; root cause of the cluster above.
- **[medium]** `diffsol-c/src/error_c.rs:24, 26` — `CString::new(bytes).unwrap_or_else(|_| CString::new("error").unwrap())` — fallback-to-`unwrap` pattern. Technically unreachable but worth a `// "error"` is a static literal containing no nul bytes comment, or just an `expect`.

### Misleading `#[allow(dead_code)]`

- **[medium]** `diffsol-c/src/jit_c.rs:24` — `#[allow(dead_code)]` on `jit_backend_to_i32`, but the function is used at `ode_c.rs:2003, 2298, 2310, 2333, 2344, 2355, 2367`. The allow attribute hides the fact that this is live code.
- **[medium]** `diffsol-c/src/jit.rs:13` — same pattern on `default_enabled_jit_backend`, which is re-exported in `lib.rs:517`.

### Incomplete features

- **[medium]** `diffsol-c/src/ode_solver_type.rs:263, 268, 389` — see "Highest priority" #5. The "single output `g` for now" comment is a real semantic limit that should be either lifted or documented in the C API surface.

---

## Examples + book

### Real bugs

- **[medium]** `examples/physics-based-battery-simulation/Cargo.toml:2` — package name typo `physics-based-batter-simulation` (missing `y`).
- **[medium]** Three orphan stub pages in `book/src/`: `solve/stopping_time.md`, `solve/stopping_event.md`, `primer/spatial_population_dynamics.md`. Each is one or two lines; none are referenced from `SUMMARY.md`.

### Cargo.toml / feature surface

- **[low]** `examples/neural-ode-weather-prediction/Cargo.toml:13` — binary requires the `onnx` feature; no default enables it, and there's no fallback. `cargo run -p neural-ode-weather-prediction` without `--features onnx` fails. Probably intentional but worth a README note.
- **[low]** `examples/mass-spring-fitting-adjoint`, `examples/predator-prey-fitting-forward`, `examples/compartmental-models-drug-delivery-sensitivities`, `examples/lorenz-attractor-diffsl-llvm` — each `src/main*.rs` constructs a `LlvmModule` but the `diffsol` dependency declaration does not enable a `diffsl-llvm*` feature by default. Same shape as above: requires explicit `--features diffsl-llvm-NN`. Intentional but undocumented for first-time runners.

### Coupling between examples and the book

- **[low]** `examples/bouncing-ball/src/main.rs:80`, `examples/pde-heat/src/main.rs:63`, and others — examples write HTML output back into `book/src/primer/images/…` using cwd-relative paths. Requires `cargo run` from the workspace root or it fails silently into the wrong location.

### Book includes — checked

All `{{#include path:start:end}}` directives across `book/src/` resolve to real files. (The agent flagged `weather_neural_ode.md:70` as a potential miss, but the path `../../../examples/neural-ode-weather-prediction/neural-ode-weather_5` is correct — the file lives at the example crate root, not under `src/`.) No broken includes found.

---

## Tests

### Float `assert_eq!` on solver/jacobian output

- **[medium]** `diffsol/src/ode_equations/mod.rs:734–737, :751–769` — `assert_eq!(jac.get_index(0, 0), -0.1)` etc., plus the same shape on the mass matrix. The current values happen to be exact in f64, but the pattern is fragile if the underlying expressions are reformulated and rounds differently. Use `approx::assert_relative_eq!` (already a dependency elsewhere).
- **[low]** `diffsol-c/tests/logistic_jit.rs:226` — `assert_eq!(y0_after, y0_before)` on f64 arrays without tolerance. Probably intentional roundtrip test, but worth a comment saying so.

### Platform-conditional `#[ignore]`

- **[medium]** `diffsol/src/ode_equations/diffsl.rs:2069–2072` — `#[cfg_attr(all(target_os = "macos", target_arch = "x86_64"), ignore = "…")]`. Silently passes on that platform. Acceptable if the reason is real and tracked, but no issue link in the source.

### Tests with no in-test assertion

- **[low]** `diffsol/src/linear_solver/mod.rs:118–137`, `diffsol/src/nonlinear_solver/mod.rs:164–184` — `test_linear_solver()` / `test_newton_cpu_*` call into a helper that does its own asserts. Functional but reading the test body alone tells you nothing.

### Test name vs. body mismatch

- **[low]** `diffsol-c/tests/logistic_jit.rs:587–599` — function name `logistic_serialization_options` but the body tests `dgdu` matrix-dimension validation (`wrong_rows`, error contains `"Expected dgdu_eval to have 1 rows"`). Either rename or split.

### Identity / vacuous assertions

- **[low]** `diffsol/src/ode_equations/mod.rs:773–793` — `test_ode_equations_statistics_new_matches_default` compares `OdeEquationsStatistics::new()` to `::default()` field-by-field. Verifies the two paths agree but not that either is correct.

### Test-module shape

- **[low]** `diffsol/src/ode_solver/mod.rs:22–1847` — `#[cfg(test)]` module spans ~1799 lines and is a *shared-helper* module (`test_ode_solver`, `test_adjoint`, etc.) with no `#[test]` of its own — all the actual tests live in `bdf.rs`, `sdirk.rs`, …. A doc comment at the top of the module would orient new readers; consider splitting `src/ode_solver/test_helpers.rs`.

### Wrapper-test repetition

- **[low]** `diffsol/src/matrix/dense_nalgebra_serial.rs:403–415` (and mirrored in `dense_faer_serial.rs:398–412`, `sparse_faer.rs:405–412`, `cuda.rs:1057–1067`) — each backend re-declares the same set of test wrappers that call into `super::super::tests::test_*` with a single concrete type. Macro-generate or accept the duplication; either way, flagged so it shows up in greps.

### Feature-gated tests with no negative complement

- **[low]** `diffsol-c/tests/logistic_jit.rs:1`, `logistic_hybrid_jit.rs:1`, `logistic_time_reset_jit.rs:1`, `logistic_stop_jit.rs:1` — `#![cfg(any(feature = "diffsl-cranelift", feature = "diffsl-llvm"))]` at file level. If neither feature is on, the file silently disappears from the test set. Standard pattern, but means CI must explicitly enable one of those features or this whole area is skipped.

### Known cosmetic

- `*_jit.rs` test files have unused-imports warnings (pre-existing, noted in earlier context). Low priority.

---

## What this audit did *not* cover

- Numerical correctness of the solvers themselves (no oracle / spec comparison).
- Performance regressions (e.g., the "TODO: remove clone" notes were flagged but not benchmarked).
- The `cuda_kernels/` directory under `diffsol/src/cuda_kernels/` (skimmed only).
- `python-diffsol/` Python-side tests / packaging.
- `population-dynamics-wasm-yew/` runtime behaviour (wasm target).
- `benchmarks/` content.
- The `book/` prose itself for technical accuracy (only structural / include checks).

Large files were sampled with `rg`-targeted reads rather than read end-to-end:
`ode_solver/bdf.rs`, `state.rs`, `builder.rs`, the 1799-line test module in
`ode_solver/mod.rs`, and `diffsol-c/src/ode_c.rs` (98K), `solve.rs` (47K),
`ode_solver_type.rs` (44K). Things that don't `rg` cleanly (subtle logic in
a long match arm, e.g.) may have been missed.

## Coverage by severity (curated counts after dedup / verification)

| Severity | Count |
|----------|-------|
| high     | 4     |
| medium   | 26    |
| low      | 27    |
