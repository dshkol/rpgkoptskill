# Profiling and performance feedback

## Contents

- Profile from the public workflow
- Select profiling tools
- Capture a defensible finding
- Give review feedback
- Raise a performance issue
- Document a performance PR

## Profile from the public workflow

Start with a reproducible exported call and a fixture shaped like user data.
Time the whole workflow, then split it into stages. Profile only the dominant
stage deeply enough to identify an actionable call path.

Keep fixture creation outside the profile unless construction is part of the
claim. Put source in a file so sampling profiles can attribute lines reliably.
Warm up or clear caches deliberately, and record which state was profiled.

For an R CPU and allocation profile:

```r
# profiling-workload.R defines `fixture` and `run_workflow()`.
source("benchmarks/profiling-workload.R")

profvis::profvis({
  result <- run_workflow(fixture)
})

tmp <- tempfile(fileext = ".out")
Rprof(tmp, memory.profiling = TRUE)
result <- run_workflow(fixture)
Rprof(NULL)
summaryRprof(tmp, memory = "both")
```

For a matched timing and allocation comparison after implementing a candidate:

```r
bench::mark(
  baseline = run_baseline(fixture),
  candidate = run_candidate(fixture),
  check = FALSE,
  min_iterations = 10
)
```

Validate complete outputs separately before setting `check = FALSE`. Prefer an
installed release or pre-change artifact for `run_baseline()`; do not make both
paths share refactored helpers. Ensure each expression receives equivalent
mutable inputs and cache state.

## Select profiling tools

- Use `profvis` for an interactive overview of R call stacks, self/total time,
  and allocations.
- Use `Rprof()` plus `summaryRprof()` for scriptable sampling evidence. Run the
  workload long enough to collect meaningful samples.
- Use `bench::mark()` for elapsed distributions, iterations, GC, and allocated
  memory; use it to compare candidates, not to locate a large call graph.
- Use `Rprofmem()`, `tracemem()`, `lobstr`, or allocation flame graphs when
  copying, object growth, or peak memory is the suspected constraint.
- Use `system.time()` for coarse stage timing when elaborate tooling would alter
  the workflow or is unavailable.
- Use backend profilers for C/C++ or Rust only after the R profile attributes
  material time below the native boundary. Include wrapper conversion time in
  the end-to-end benchmark.
- For databases, APIs, files, or browsers, capture queries/requests, bytes,
  server or disk time, serialization, browser parse/render, and R-side time
  separately. An R sampler cannot explain time spent outside R.

Do not treat sampling percentages as elapsed improvements. A function can rise
from 20% to 40% of a faster total while becoming faster in absolute terms.

## Capture a defensible finding

For each material hotspot, record:

1. exported workload, fixture shape, and user impact;
2. environment, versions, thread count, and cold/warm state;
3. end-to-end baseline distribution and resource metrics;
4. profile call path, self/total attribution, call count, and allocation;
5. observed scaling across the dimensions that drive cost;
6. mechanism supported by evidence;
7. proposed smallest compatible change and its risks;
8. exact command or script that reproduces the evidence.

Label statements as:

- **Observed:** directly measured, such as 62% of samples below a grouped call.
- **Inferred:** an explanation consistent with the evidence, such as per-group
  dispatch dominating many tiny groups.
- **Proposed:** an unmeasured candidate, such as replacing dispatch with one
  boundary-masked vector operation.
- **Demonstrated:** an implemented candidate with matched before/after results
  and semantic validation.

Do not call a proposal an improvement until the candidate benchmark has run.

## Give review feedback

Make performance review comments specific and actionable. Include the relevant
path or call, realistic workload, evidence, consequence, and requested next
step. Distinguish a correctness or severe regression blocker from an optional
optimization suggestion.

Use this compact structure:

```text
Finding: [measured behavior and workload]
Evidence: [profile/benchmark result and reproduction command]
Why it matters: [absolute user impact, scaling, memory, I/O, or regression]
Suggested action: [smallest architecture-compatible change or benchmark needed]
Acceptance: [semantic checks and measured threshold]
```

Do not request a rewrite from visual inspection alone. If a PR claims improved
performance but supplies no comparable benchmark, request a reproducible
before/after result rather than guessing whether the code is faster.

After an accepted optimization, re-profile the complete workflow once. The
first win can expose a formerly secondary hotspot; pursue it only when the
remaining absolute cost and implementation complexity justify another change.
During iterative review, re-check comments against the latest commit and rerun
the benchmark whenever a fix changes the measured path.

## Raise a performance issue

An issue may report a confirmed bottleneck before a fix exists. Include:

- concise problem and affected public workflow;
- package/dependency versions and environment;
- minimal reproducible fixture or generator with safe data;
- baseline distribution and resource metrics;
- profile artifact or summarized call path;
- scaling table or at least two informative sizes;
- observed versus inferred statements;
- likely opportunity and architectural constraints;
- acceptance criteria for a future fix;
- reproduction command and any benchmark script path.

Avoid prescribing C++, Rust, parallelism, or a new dependency unless evidence
and package context justify raising it as an option. Never include credentials,
private data, machine-specific paths, or huge generated artifacts.

## Document a performance PR

A performance PR must show actual results for the implemented diff. Include:

- problem and profile evidence that selected the target;
- baseline version/commit and candidate commit;
- behavior-preserving mechanism, or clearly authorized behavior change;
- benchmark fixture, exact command, environment, repetitions, and cache/thread
  policy;
- absolute and relative before/after results across relevant workload sizes;
- time plus memory, I/O, output size, or other affected resources;
- complete-object equivalence method and edge cases;
- focused tests, full tests, and package-check results;
- complexity, dependency, portability, and maintenance trade-offs;
- benchmark artifact location and any published documentation updated.

Report regressions and neutral cases alongside wins. If the candidate failed the
predeclared materiality threshold, do not present it as a successful performance
PR. Keep an unsuccessful prototype out of production unless it has a separate,
documented benefit.

Drafting feedback does not authorize posting it. Submit reviews, issues, or PRs
only when the user explicitly requests that external action.
