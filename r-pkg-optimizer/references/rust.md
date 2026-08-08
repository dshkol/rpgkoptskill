# Existing R and Rust packages

Use this guidance when an extendr, rextendr, savvy, or raw Rust backend already
exists, or after explicit user authorization. Do not propose Rust merely as a
synonym for speed.

## Find the actual boundary cost

- Measure R wrapper time, R-object conversion, allocation, native kernel time,
  and result materialization separately.
- Batch scalar calls and keep work in Rust long enough to amortize conversion,
  without hiding an inspectable public R representation users rely on.
- Borrow or view input data when the framework and lifetime rules make it safe
  (numeric vectors can often be viewed as slices without copying); otherwise
  make copying explicit and measure it.
- Strings are the expensive boundary: R's `CHARSXP` pool does not map to Rust
  `String` without conversion and allocation per element. Batch string work,
  convert once, and prefer operating on bytes or interned indices when the
  contract allows.
- Preallocate `Vec` and output structures when size is known. Avoid repeated
  string conversion, cloning, and R callbacks inside hot iteration.
- Apply the same length-1-or-length-n parameter recycling contract as C
  backends when vectorizing per-element parameters (see
  [native-cpp.md](native-cpp.md)); reject incompatible lengths.
- Build and benchmark release/performance profiles, not debug builds.

## Preserve package viability

- Keep generated wrappers generated and use the package's established tooling.
- Respect R's single-threaded API: do not call into R or manipulate R objects
  from worker threads.
- Catch or convert errors according to the binding framework; never allow a
  panic to cross the R FFI boundary.
- Audit crate features, toolchain requirements, vendoring, offline builds,
  WebR/WASM claims, CRAN policies, and supported platforms.
- Parallelize only pure native computation with enough grain to repay scheduling
  and copying; make thread count controllable and avoid oversubscribing BLAS or
  R-level parallelism.

## Validate

Test conversions for missing and special values, integer widths, strings and
encodings, zero-length data, errors, interrupts where supported, determinism,
and cross-platform floating-point behavior. Compare full R-visible output and
conditions against the baseline.

Adding Rust to an R-only package must pass the escalation test in `SKILL.md` and
requires user direction before implementation.
