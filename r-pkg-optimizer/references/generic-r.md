# Generic R optimization

Apply these patterns regardless of package style, after profiling identifies a
relevant hot path. Every pattern here is idiom-neutral: it changes internal
execution, not the package's interface or flavor.

## Estimate before prototyping

Use back-of-the-envelope reasoning to reject implausible candidates before
implementing them, then measure the survivors. Estimate call count, input and
intermediate sizes, asymptotic growth, bytes crossing representation boundaries,
and whether setup or dispatch repeats per element. Do not embed generic timing
constants in a claim: R version, method dispatch, object classes, hardware,
dependencies, and cache state can change them substantially.

Two recurring hypotheses are worth testing:

- Per-element R dispatch can dominate when the body performs only trivial work.
  Batching removes repeated dispatch, but it is useful only at sufficient call
  counts.
- Repeated data-frame construction, slicing, or binding can dominate the vector
  operation being performed. Measure the public workflow before replacing it.

## Remove and reuse work

- Delete calculations whose results are discarded and branches made
  unreachable by earlier validation. Read hot functions for dead work before
  optimizing live work; deletion is often the lowest-complexity improvement.
- Use `requireNamespace(pkg, quietly = TRUE)` for a package-presence test; do
  not scan the entire library with `installed.packages()` on any path.
- Grep for `TODO`/`FIXME` before profiling. Maintainers often document known
  waste ("this round-trips through encode/decode") that profiling would only
  rediscover.
- Hoist invariant parsing, name resolution, regex compilation, column
  selection, coercion, and metadata lookup out of loops.
- Precompute stable properties once (at `.onLoad`, first use, or object
  construction) when they are re-derived on every call; defer expensive
  properties that most calls never read.

## Improve scaling before constants

- Name what the cost scales with and what the *answer* scales with. A mismatch
  (cost tracks a large intermediate; answer is small) is the biggest available
  win and converts impractical cases into trivial ones. Micro-tuning the loop
  over the intermediate never can.
- Traverse graphs and hierarchies over atomic key vectors; join back to the
  rich data frame once at the end. The canonical anti-pattern filters and
  `rbind`s data frames at every level, re-processing all accumulated rows:

  ```r
  # BFS over atomic keys; `edges` has `parent` and `child` vectors.
  seen <- character(0)
  frontier <- unique(roots)
  while (length(frontier) > 0) {
    current <- frontier[!(frontier %in% seen)]
    if (!length(current)) break
    seen <- c(seen, current)
    candidates <- edges$child[edges$parent %in% current]
    frontier <- unique(candidates[!(candidates %in% seen)])
  }
  result <- all_rows[match(seen, all_rows$key), ]
  ```

  `seen` accumulates in discovery order, which can reproduce keep-first
  ordering. Verify ordering equivalence, not just set equivalence.
- Prune expensive pairwise metrics with a cheap necessary condition that
  provably cannot discard a valid result: length difference lower-bounds edit
  distance, bounding boxes lower-bound geometric distance, metadata bounds
  deserialization. Because the bound is provable, no output re-verification is
  needed beyond a threshold audit.
- Batch calls when setup, dispatch, parsing, or conversion repeats. Check
  whether the workhorse function already accepts a vector or matrix before
  writing a loop over elements, files, groups, or periods — one vectorized
  `file.info(paths)`, `xml_find_first(nodeset, ...)`, or matrix-accepting
  linear-algebra call replaces thousands of scalar calls.

## Reduce allocation and copying

- Preallocate known-size atomic vectors and lists. Do not grow them with
  `c()`, `append()`, or repeated `rbind()` in a large hot loop. Defer fixing
  asymptotic problems that remain immaterial at the realistic upper bound; set
  a revisit threshold instead of adding speculative machinery.
- Avoid slicing data frames row-wise in a measured hot loop. Resolve columns
  once, validate the column vectors together, then construct each output
  directly from column values. Compare the complete constructed object with
  the baseline.
- Sweep for double-scan idioms; compute the mask once:

  ```r
  # Avoid a guard scan followed by a second duplicate scan.
  dup <- duplicated(x)
  if (any(dup)) x <- x[!dup]
  ```

  Prefer logical subsetting over `x[-which(...)]`; `which()` allocates an
  index vector for no benefit. Same family: `any(cond)` then `which(cond)`,
  `is.na(x)` computed twice, `sort()` then `order()`.
- Use `vapply(..., FUN.VALUE)` over `lapply()`/`sapply()` when the consumer
  needs an atomic vector — building a list and coercing it on every downstream
  call is a recurring tax paid at the wrong site.
- Match the container to the consumer: if `grep()`/`regmatches()` receive the
  object, build it as character once, not as a list coerced per call. Match
  storage mode too — repeated character/factor/matrix/tibble coercions in hot
  paths are silent copies.
- Replace per-element `strsplit()`/`substr()` extraction with one vectorized
  `sub()`/`regmatches()` call; prefer `startsWith()`/`endsWith()` and
  `fixed = TRUE` (or `perl = TRUE` for complex patterns) over default-regex
  matching in hot string paths.
- Inspect GC-heavy profiles for many short-lived objects and repeated copies.
  Use `tracemem()` or `lobstr::obj_size()` when copy behavior is unclear;
  modifying a data-frame cell in a loop copies columns each iteration —
  operate on extracted vectors and assign back once.

## Fast paths and caching

- Add a fast path when a cheap test identifies a common case whose full
  machinery is unnecessary (scalar parameter, already-sorted input,
  no-missing-values input). Keep the general path as the fallback and test
  both against each other.
- When the same derived object is recomputed within a session, cache it in a
  package-level environment. See [io-caching.md](io-caching.md) for the
  layered-cache contract — including why a disk cache is not a fast cache and
  how refresh flags must reach every layer.

## Respect R semantics

Vectorized rewrites must handle empty vectors, missing values, recycling,
integer versus double output, names, dimensions, factors, time zones, ALTREP,
and attributes. Avoid relying on undocumented internals for small gains.

Byte compilation and JIT may help loop-heavy R code, but measure the installed
package path. Do not replace a clear, cold, call-once function for a
compile-time win users will not notice.

## Read call patterns, not isolated lines

Map the call graph before ranking targets. A modest helper called repeatedly by
many exported workflows can outweigh a visibly slow function called rarely;
frequency and end-to-end impact matter alongside single-call latency.

Conversely, functions operating on already-aggregated small objects (k-by-k
matrices, short summaries) are microseconds by construction — measure them
once so slowness is a decision rather than a surprise, then usually stop.

Preserve readability in non-hot code. Make optimized private helpers state and
check any stronger requirements for sorting, uniqueness, shape, or ownership —
a sorting precondition left as a comment is a latent bug for the next caller.
