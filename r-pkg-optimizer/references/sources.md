# Authoritative sources and further reading

Use current primary documentation when tool behavior, package internals, CRAN
policy, or native build requirements affect an optimization. Treat examples and
timings in any source as workload-specific evidence, not reusable constants.

## Core R profiling and package guidance

- R Core, [`Rprof()`](https://stat.ethz.ch/R-manual/R-devel/library/utils/html/Rprof.html)
  and [`summaryRprof()`](https://stat.ethz.ch/R-manual/R-devel/library/utils/html/summaryRprof.html):
  sampling events, self/total attribution, line profiling, memory summaries,
  and platform limitations.
- R Core, [`Rprofmem()`](https://stat.ethz.ch/R-manual/R-devel/library/utils/html/Rprofmem.html):
  allocation profiling and its interpretation.
- R Core, [Writing R Extensions](https://cran.r-project.org/doc/manuals/r-release/R-exts.html):
  profiling, compiled code, portability, registration, and package checks.
- Hadley Wickham, [Advanced R: Measuring performance](https://adv-r.hadley.nz/perf-measure.html):
  profiling and benchmark design for R code.

## Benchmarking and general optimization

- `bench`, [`mark()`](https://bench.r-lib.org/reference/mark.html) and
  [`press()`](https://bench.r-lib.org/reference/press.html): timing,
  allocations, garbage collection, result checking, and parameter grids.
- `profvis`, [`profvis()`](https://profvis.r-lib.org/reference/profvis.html):
  interactive visualization of R profiling data.
- Abseil, [Performance Hints](https://abseil.io/fast/hints.html): estimation,
  batching, precomputation, allocation reduction, fast paths, caching, and
  evidence-driven optimization. Translate the mechanisms to R; do not copy
  C++-specific advice without checking R semantics.

## Architecture-specific documentation

- `data.table`, [Benchmarking data.table](https://cran.r-project.org/web/packages/data.table/vignettes/datatable-benchmarking.html),
  [optimisations](https://rdatatable.gitlab.io/data.table/reference/datatable-optimize.html),
  [joins](https://rdatatable.gitlab.io/data.table/articles/datatable-joins.html),
  [`rowid()`](https://rdatatable.gitlab.io/data.table/reference/rowid.html), and
  [`rleid()`](https://rdatatable.gitlab.io/data.table/reference/rleid.html):
  benchmark traps, optimized query shapes, specialized joins, and group/run
  identifiers.
- `sf`, [geometric binary predicates](https://r-spatial.github.io/sf/reference/geos_binary_pred.html):
  spatial indices, prepared geometries, sparse output, and backend behavior.
- `rextendr`, [Using Rust code in R packages](https://extendr.github.io/rextendr/articles/package.html):
  Rust package layout, generated wrappers, and build workflow.

Consult the current documentation for the package's actual C/C++ or Rust
binding framework and supported versions before editing native code.
