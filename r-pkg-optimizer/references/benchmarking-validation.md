# Benchmarking and validation

## Contents

- Benchmark contract
- Fixture design
- Measurement hygiene
- Independent equivalence harness
- Offline benchmarks for API packages
- Performance evidence
- Before/after decision rule
- Package integration

## Benchmark contract

Write down before editing:

- public operation and user scenario;
- baseline version or commit;
- typical and stress sizes, distributions, cardinalities, sparsity, and order;
- metric: latency, throughput, allocation, peak memory, output bytes, requests,
  startup, build time, or a combination;
- cold/warm cache and process state;
- required semantic equivalence and numerical tolerance;
- minimum practically meaningful improvement.

Use end-to-end measurement to rank user impact and stage-level measurement to
locate the cost. A microbenchmark answers only a narrow mechanistic question.

## Fixture design

- Make synthetic data preserve the dimensions that control cost: group count
  and size, key cardinality, string length, geometry complexity, spatial
  extent, matrix shape, missingness, or serialized depth. Row counts drive
  hash and join behavior; string lengths drive fuzzy matching. Structure the
  fixture like real data (many units, several periods, realistic ordering),
  not like a convenient array.
- Use real or anonymized fixtures when synthetic structure is not credible.
- Mock network and clock boundaries. Exercise the real public path above the
  mock.
- Record deterministic seeds, dependency versions, R version, platform, thread
  count, and relevant options.
- Keep setup outside the timed expression unless setup is part of user cost.

## Measurement hygiene

- Run enough iterations for stable distributions; report medians and ranges or
  quantiles, not the best run.
- Randomize candidate order when drift matters. Compare in the same session
  when isolation allows it, or use matched clean sessions.
- Warm compilation and deliberate caches consistently. Report cold and warm
  timings separately.
- Account for GC and allocations. Faster code that doubles peak memory may be a
  regression for the target workload.
- Avoid accidental mutation between iterations. `data.table` and environments
  need fresh inputs or a deliberate copy policy.
- Control multithreaded libraries. Record `data.table::getDTthreads()` and BLAS
  thread settings where relevant.
- For I/O, distinguish OS page cache, application cache, decompression, parsing,
  and downstream work.

## Independent equivalence harness

Prefer an installed released package in an isolated library over two functions
in one refactored tree — it measures what users actually ran, avoids
`load_all()` overhead asymmetries, and cannot share newly refactored helpers
with the candidate:

```sh
R CMD INSTALL -l /tmp/baseline-lib pkg_x.y.z.tar.gz   # released tarball or tag
```

```r
library(pkg, lib.loc = "/tmp/baseline-lib")
# Run representative and adversarial public calls; saveRDS() every component
# of every result object as the baseline. Then run the development tree on
# identical inputs and compare component by component.
```

Run identical cases and save:

- complete return values;
- class, attributes, names, dimensions, ordering, grouping, and geometry;
- warnings, messages, errors, and visibility;
- files, cache entries, mutation, RNG effects, and external calls.

Compare with strict `testthat` expectations, `all.equal()`, or
`waldo::compare()`. Expect iteration: reaching exact equivalence has taken
multiple rounds in practice because of invisible semantic differences —
grouped round-trips stripping attributes, dedup loops implying keep-first
ordering. When replacing an accumulate-and-dedup loop, prove **ordering**
equivalence across the full option matrix on randomized inputs; set
equivalence is not enough. For floating-point algorithms, justify tolerance
from the method rather than choosing one that makes the test pass. Compare
randomized cases when the option matrix is large, while retaining
deterministic regression cases for failures.

Benchmark correctness and speed separately so comparison work is not timed.
Time baseline and candidate in one session via `library(pkg, lib.loc = ...)`
against the same fixture.

## Offline benchmarks for API packages

For a package wrapping a remote source, honest benchmarks must not hit the
network:

- generate mock fixtures sized like real responses;
- stub the data-source function with
  `testthat::with_mocked_bindings(..., .package = "pkg")` so public functions
  run their real code paths above the mock;
- fabricate the on-disk cache file directly in `tempdir()` to benchmark the
  cache-read path;
- record the before/after table and environment in the script header, and run
  the same script against an install or worktree of the old version to
  reproduce the comparison.

Measure cold (remote-equivalent), warm-disk, and warm-memory paths
separately, and count requests and bytes as well as elapsed time. Do not
publish speedups dominated by an unreported pre-warmed cache.

## Performance evidence

Vary each claimed scaling driver and include it in the result table, alongside
the intermediate driver that explains the mechanism—not just the outcome. For
a spatial covering, for example, include point count, spread, precision, and
candidate-cell count with baseline and candidate results.

Report absolute before/after values plus ratios. Mark baseline cases that are
impractical explicitly rather than silently omitting them — "the old path
cannot run this at all" is often the strongest result. Record machine and R
version, and state that absolute times vary across machines while scaling
behavior does not. Explain the mechanism: fewer candidate cells, one batched
call instead of thousands of dispatches, or no repeated deserialization.

Avoid performance tests with tight wall-clock thresholds in routine package
checks. Prefer structural guards such as bounded call counts, no repeated I/O,
linear-size intermediates, or benchmark scripts reviewed manually or in a
controlled performance job.

## Before/after decision rule

After implementing a candidate, execute the matched benchmark; do not stop at
tests, static reasoning, or a profiler showing that the edited lines were hot.
Compare the baseline artifact and candidate under the same fixture, options,
cache policy, thread policy, and measurement process.

Report a table containing at least:

| Workload and size | Baseline | Candidate | Absolute change | Relative change | Other resource change |
|---|---:|---:|---:|---:|---:|

Calculate speedup consistently as `baseline / candidate` for elapsed time and
state the convention. Also report the absolute saving: a 10x speedup from 10 µs
to 1 µs may be irrelevant, while a 15% saving in a repeated minute-long stage
may be valuable. For throughput, memory, or bytes, name the direction explicitly.

Keep the optimization only when it produces a repeatable, practically material
improvement on the target workload without unacceptable regressions elsewhere.
If it does not, revert the candidate or present it explicitly as an unsuccessful
experiment. Never round noise into a positive claim or report only the best
input. Include neutral and regressed cases.

## Package integration

Put reproducible scripts under `benchmarks/`; add the directory to
`.Rbuildignore` (an untracked top-level directory otherwise triggers an
R CMD check NOTE for a non-standard file at top level). Include fixture
generation, environment notes, exact commands, and compact results — saved
`.rds` results and plots referenced from the PR let reviewers re-run the
claim. Do not commit large generated data unless justified.

If performance claims live in rendered documentation, tie re-rendering into
the change itself, not the follow-up: gate expensive chunks on
`identical(Sys.getenv("IN_PKGDOWN"), "true")` so they skip during
R CMD check, and re-render committed `docs/` in the same change when numbers
go stale — a published stale benchmark misrepresents the package.

Keep correctness fixes found during the optimization pass in separate commits
from behavior-preserving performance changes, so the perf diffs stay
reviewable and a revert of one does not drag the other. An optimization pass
is an effective bug hunt — budget a quick-fixes phase at the start; it merges
fast and builds maintainer trust.

Useful starting tools are `bench`, `profvis`, `Rprof()`/`summaryRprof()`,
`tracemem()`, `lobstr`, backend profilers, and OS-level memory measurement. Use
the smallest tool that can answer the current question. See
[profiling-feedback.md](profiling-feedback.md) for profiling commands and
evidence capture.
