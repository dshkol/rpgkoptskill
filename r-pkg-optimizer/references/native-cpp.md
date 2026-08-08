# Existing C, C++, Rcpp, and cpp11 backends

Use this guidance only when the package already has native code or the user has
authorized adding it. First determine whether time is in the native kernel, R
wrapper, conversion, allocation, or repeated boundary crossing. `Rprof()`
cannot see inside native code — a single opaque `.Call` frame dominating the
profile means switching to `perf` (Linux) or Instruments (macOS) for
attribution inside the kernel.

## Optimize the boundary first

- Batch many scalar calls into one vector or matrix call when conversion and
  `.Call` dispatch dominate.
- Pass compact atomic inputs; avoid R-to-C++ data-frame reconstruction when
  columns suffice.
- Allocate output and scratch buffers once. Size reusable scratch space to the
  maximum required value outside the loop.
- Vectorize per-element parameters inside an existing native loop instead of
  driving scalar calls from an R-level `vapply()`. Preserve R's documented
  length-one-or-length-n contract and reject incompatible lengths:

  ```c
  const int nk = LENGTH(k_arg);
  const int *kp = INTEGER_RO(k_arg);
  /* scratch sized once to the largest requested value, outside the loop */
  char *buf = (char *) R_alloc(kmax + 1, sizeof(char));
  for (int i = 0; i < n; i++) {
    int k = kp[nk == 1 ? 0 : i];   /* recycle length-1, else per-element */
    ...
  }
  ```

  The recycling branch keeps the change backward compatible with no R-side
  dispatch.
- Avoid native-to-R callbacks in hot loops.

## Native hot loops

- Select algorithms and contiguous representations before micro-tuning.
- Avoid repeated allocation, parsing, hashing, virtual dispatch, and duplicate
  lookup. Hoist invariants and reserve capacity where size is known.
- Use read-only accessors and `const` inputs where supported (`INTEGER_RO`,
  `REAL_RO`) — they cost nothing and catch accidental mutation of R-owned
  memory.
- Do not mutate R objects unless the interface explicitly owns a safe mutable
  object.
- Use portable R allocation and protection rules. In C, prefer `R_alloc` for
  temporary request-lifetime storage over C variable-length arrays — VLAs are
  a portability problem reviewers will flag, and `R_alloc` memory is reclaimed
  on error without cleanup code.
- In Rcpp, preallocate output vectors, avoid `wrap`/`as` churn inside loops,
  and keep object construction out of the hot path.
- Check interrupts in sufficiently long loops without checking so frequently
  that the check dominates.
- Respect CRAN compiler flags, sanitizers, multiple architectures, and the
  package's C/C++ standard. Users compile with their own flags — do not depend
  on `-O3`-specific behavior, and never ship `Makevars` overriding
  optimization levels.

## Validate

Test zero-length inputs, `NA`/`NaN`/`Inf`, integer overflow, encoding, recycling,
interrupts, errors after allocation, and platform-dependent numeric behavior.
Bounds-check parameters in the native code — a missing check that silently
produces wrong answers for out-of-range input is the classic bug found while
reading native hot paths. Run compiled-code checks and sanitizers appropriate
to the package. Benchmark installed optimized builds; development flags can
invalidate conclusions.

Do not migrate Rcpp to cpp11, C++ to Rust, or vice versa as a performance tweak.
Such a migration is an architecture project requiring explicit direction and
independent justification.
