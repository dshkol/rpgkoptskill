---
name: r-pkg-optimizer
description: Profile, diagnose, optimize, validate, and communicate performance work in existing R packages while preserving their public API, semantics, dependency posture, and chosen implementation style. Use for slow R functions or workflows; CPU, allocation, memory, I/O, cache, serialization, startup, or build-time problems; benchmark design; performance regressions; performance reviews, pull requests, or issues; base R, tidyverse, data.table, Rcpp/C/C++, Rust, geospatial, API, HTML widget, JavaScript, and numerical packages; or deciding whether a native backend is justified.
---

# R Package Optimizer

Improve measured user workflows without turning performance work into an
unrequested redesign.

## Governing principles

- Preserve the package's architecture and idiom. Optimize tidy code as tidy
  code, `data.table` code as `data.table`, and an existing native backend in
  its native language.
- Treat the public API, values, classes, attributes, ordering, missingness,
  warnings, errors, side effects, and mutation behavior as the compatibility
  contract unless the user authorizes a change.
- Measure before editing. Optimize end-to-end workflows first, then profile the
  dominant stage. Do not infer a bottleneck from syntax or reputation.
- Require an executed before/after benchmark for every implemented performance
  claim. Never describe an estimate, profiler percentage, or microbenchmark of
  a different operation as an actual user-workflow improvement.
- Prefer, in order: delete unused work; avoid repeated work; choose a better
  algorithm; batch or vectorize; reduce conversions, copies, and allocations;
  tune implementation details.
- Charge every optimization for complexity, dependencies, portability, build
  burden, and maintenance. Reject changes whose realistic benefit does not pay
  that bill.
- Never introduce C++, Rust, parallelism, a cache, or a new dependency solely
  because it might be faster. See the escalation policy below.
- Keep correctness fixes found during profiling distinct from behavior-
  preserving performance changes.

## Resolve authority and package context

Read `AGENTS.md`, `DESCRIPTION`, `NAMESPACE`, `R/`, `src/`, `tests/`, benchmark
artifacts, vignettes, and CI configuration before proposing an implementation.
Infer the package's design from its dependencies and code, not its task label.

Classify the request:

- **Review, explain, or diagnose:** inspect and report evidence; do not edit.
- **Optimize, fix, or implement:** make the smallest supported change and
  validate it.
- **Explore:** prototype outside production paths and report trade-offs unless
  the user asks to land a candidate.

Record the relevant context:

- exported workflow and user-visible pain;
- typical and upper-bound workload shapes;
- R version floor and supported platforms;
- dominant idiom: base R, tidyverse, `data.table`, native, or mixed;
- existing mutability, caching, concurrency, and dependency policies;
- whether inputs are in-memory, remote, on disk, spatial, serialized, or passed
  across R/native/browser boundaries.

## Core workflow

### 1. Define the claim before measuring

State the workload, scale dimensions, metric, baseline, compatibility contract,
and success threshold. Prefer a public call with realistic data over an
isolated helper. Include cold/warm state, thread count, cache state, and network
policy when they can change the result.

Read [benchmarking-validation.md](references/benchmarking-validation.md) before
designing, publishing, or relying on benchmarks.

### 2. Establish an independent correctness baseline

Use the latest released version, relevant tag, or pre-change commit installed
in an isolated library when feasible. Save complete outputs and conditions for
representative and adversarial inputs. Do not construct both baseline and
candidate through newly shared helpers.

### 3. Measure the workflow in stages

Split construction, transformation, validation, I/O, serialization, and output
materialization where applicable. Measure elapsed distributions and relevant
resource quantities such as allocations, peak memory, object size, bytes read,
serialized size, requests, candidate counts, or compile time.

Warm up JIT-compiled or cached paths deliberately. Benchmark cold and warm
behavior separately when users experience both. Avoid network benchmarks;
mock the remote boundary with realistic fixtures.

### 4. Profile the dominant stage

Use `profvis`, `Rprof()`, `summaryRprof()`, `bench`, allocation tracing, or
backend-specific profilers in proportion to the problem. Read profiles by self
time, total time, call count, allocation, and scaling. A slow outer wrapper may
only be evaluating an expensive argument.

Name what the cost scales with. Compare it with what the answer scales with.
When they differ—for example bounding-box cells versus input points—look for a
new formulation before tuning the loop.

Read [profiling-feedback.md](references/profiling-feedback.md) for tool
selection, profiling commands, evidence capture, and review/issue/PR formats.

### 5. Rank opportunities

Prioritize opportunities by:

`expected user impact × confidence × frequency ÷ complexity and risk`

Search in this order:

1. dead, unreachable, discarded, or duplicate work;
2. repeated I/O, parsing, serialization, conversion, validation, or lookup;
3. large intermediates whose size exceeds the result;
4. worse asymptotic algorithms or data structures;
5. scalar/per-group dispatch that can be batched;
6. copies, coercions, allocations, and representation round-trips;
7. hot-loop implementation details.

Use cheap, provably safe rejection tests before expensive work: length bounds
before edit distance, bounding boxes before exact geometry, metadata before
deserialization, and structural checks before matrix reshaping.

### 6. Prototype the smallest viable change

Change one mechanism at a time. Move invariant work out of loops, preallocate,
operate on atomic keys, and join rich objects back once. Prefer an existing
vectorized or matrix-aware backend operation over hand-written iteration.

Do not weaken validation merely to improve a benchmark. Move redundant checks
to a trustworthy boundary only when the same contract remains enforced.

### 7. Validate semantics and scaling

Compare complete objects and conditions, including classes, attributes, names,
row and column order, grouping, geometry, factors, time zones, missing values,
numeric tolerances, mutation, files, cache invalidation, warnings, and errors.

Test zero, singleton, typical, large, missing, duplicated, unsorted, malformed,
and structurally irregular cases as relevant. Benchmark the same fixture and
session policy against the independent baseline. Vary the dimensions claimed
to drive scaling; do not publish a universal multiplier from one input.

Run focused tests while iterating, then the complete tests and package check.
Keep unstable timing assertions out of `R CMD check`; test invariants and use
separate benchmark scripts for performance evidence.

### 8. Report the result honestly

Report workload, versions, environment, repetitions, summary statistic,
correctness method, absolute timings, relative change, and residual trade-offs.
State whether the change improves latency, throughput, memory, file size,
startup, or another metric; do not substitute one for another.

Check benchmark artifacts into `benchmarks/` when the claim should be
reproducible, and add that directory to `.Rbuildignore`. Re-render committed
performance articles when their numbers change.

When giving review feedback or drafting an issue or PR, connect every claim to
a reproducible workload and separate observation from inference. Include the
profile attribution, before/after results when a candidate exists, semantic
validation, trade-offs, and exact reproduction command. Do not post a review,
open an issue, or create a PR unless the user authorized that external action.

### 9. Preserve recommendation history

When an investigation produces a recommendation for an external package, read
[recommendation-history.md](references/recommendation-history.md). Check for an
existing record before creating one, distinguish a private draft from feedback
actually sent to maintainers, and update the same record when the package or
maintainers respond.

Capture the evidence, outcome, negative results, and transferable learning.
Use an explicitly designated portfolio ledger; do not add bookkeeping files to
the package under review or contact maintainers merely to complete the record.
If no writable ledger is available, return a ready-to-file record with the
work product instead of silently losing the history.

## Architecture routing

Read only the references matching the package and measured bottleneck:

- **All packages:** [generic-r.md](references/generic-r.md)
- **Tidyverse/tibble internals:** [tidyverse.md](references/tidyverse.md)
- **`data.table`:** [data-table.md](references/data-table.md)
- **Existing C/C++/Rcpp/cpp11 backend:** [native-cpp.md](references/native-cpp.md)
- **Existing Rust/extendr backend:** [rust.md](references/rust.md)
- **`sf`, `s2`, `terra`, raster/vector GIS:**
  [geospatial.md](references/geospatial.md)
- **HTML widgets, JSON, JavaScript, browser boundaries:**
  [web-serialization.md](references/web-serialization.md)
- **Remote APIs, files, databases, and caches:**
  [io-caching.md](references/io-caching.md)
- **Matrix, statistical, numerical, or parallel workloads:**
  [numerical-parallel.md](references/numerical-parallel.md)
- **Profiling and performance feedback in reviews, issues, and PRs:**
  [profiling-feedback.md](references/profiling-feedback.md)
- **Authoritative sources and further reading:**
  [sources.md](references/sources.md)

For mixed packages, preserve the user-facing layer and optimize at the layer
where the profile attributes cost. Crossing a boundary is itself a candidate
cost: R to native, R to database, geometry conversions, and R objects to JSON.

## Native-backend escalation policy

Optimize an existing C, C++, Rcpp, cpp11, or Rust backend when the hot path is
already there. Do not migrate one native backend to another without explicit
direction.

For an R-only package, exhaust measured improvements compatible with its
current design first. Merely flag—not implement—a native backend only when all
of these hold:

- a stable, compute-bound kernel remains dominant on realistic workloads;
- the operation has enough work per call to amortize boundary conversion;
- profiling and an R-level prototype show the likely ceiling;
- the kernel has a crisp, independently testable contract;
- expected gains materially affect users;
- platform, CRAN, toolchain, dependency, debugging, and maintainer costs are
  acceptable.

Present the evidence, candidate boundary, expected benefit, and maintenance
cost, then ask for user direction. Prefer a specialized existing dependency
when it fits the package's dependency policy and avoids owning native code.

## Fast review searches

Use these as leads, never as proof:

```sh
rg -n "for \(|while \(|lapply|sapply|group_(by|modify|map)|rowwise" R src
rg -n "rbind|bind_rows|c\(|append|which\(|anyDuplicated|installed.packages" R
rg -n "readRDS|load\(|readLines|fromJSON|toJSON|st_(transform|as_sf)|as\." R
rg -n "TODO|FIXME|slow|cache|serialize|deserialize|copy|alloc" R src
```

Interpret every hit in its call pattern and workload. Readable non-hot code is
usually already fast enough.
