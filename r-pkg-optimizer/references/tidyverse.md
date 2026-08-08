# Tidyverse and tibble packages

Preserve a tidy public interface unless profiling shows the representation
boundary itself dominates. Diagnose specific grouped dispatch, reconstruction,
copy, join, pivot, or row-wise costs; "tidyverse overhead" is not a diagnosis.

Profile before believing any hypothesis about tidy overhead. A comparison with
a matrix-native implementation does not show whether the public tibble
interface, per-group dispatch, reconstruction, or underlying algorithm is
responsible for the difference.

## Common opportunities

- **Many tiny groups are the canonical hot spot.** `group_by(unit) |>
  mutate(lead(x))` repeats grouping, slicing, reconstruction, and attribute
  work once per group. When profiles show dispatch dominating many small
  groups, test a sort-once whole-column operation (below).
- Replace `rowwise()` or one-row slicing with column-wise coercion and direct
  construction of final row objects.
- Check whether the workhorse accepts vectors or matrices before reaching for
  `group_modify()`, `group_map()`, or one call per period. Spatial and
  linear-algebra dependencies often accept a matrix and process every column
  in one call (e.g. `spdep::lag.listw()`); reshaping a balanced panel to a
  units-by-periods matrix collapses the loop. Batching is safe only after
  validating the reshape: every period complete, identifiers aligned with any
  external weights object, flattening restoring intended order. Reject
  irregular panels rather than allowing silent recycling.
- Select columns and resolve tidy-eval expressions once outside repeated work.
- Avoid `bind_rows()` plus `distinct()` on an expanding accumulator.
  Accumulate atomic keys or list pieces and assemble once (see the traversal
  pattern in [generic-r.md](generic-r.md)).
- Repeated `pivot_longer()`/`pivot_wider()` on small objects is often
  immaterial. Measure it once; even a large relative difference may remain too
  small in absolute terms to justify a less readable extraction path.
- In internal hot paths, `vctrs::new_data_frame()` or
  `tibble::new_tibble(..., nrow = n)` skips validation that
  `tibble()`/`as_tibble()` repeat; use only where inputs are already
  guaranteed valid.

## Boundary-masked shift pattern

For data sorted by (group, time), a within-group `lead` is one whole-vector
shift plus masking at group boundaries:

```r
# Replaces df |> group_by(grp) |> mutate(y = lead(x)) |> ungroup()
# for data already sorted by (grp, time).
n <- length(x)
out <- x[c(seq_len(n)[-1L], NA_integer_)]              # shift whole vector once
differs <- (grp[-n] != grp[-1L]) |
  (is.na(grp[-n]) != is.na(grp[-1L]))                  # adjacent group change
differs[is.na(differs)] <- FALSE
out[c(differs, TRUE)] <- NA                            # mask boundaries + tail
```

The same shape generalizes to lags, cumulative counters, and first/last flags.
The vectorized helper relies on stronger preconditions than the grouped
pipeline — make the sorting requirement explicit at the caller boundary and
test: zero rows, one row, one group, singleton and uneven groups, missing
group identifiers, duplicated unit-period pairs, and unsorted input per the
documented contract.

## Semantic traps

Grouped operations can change grouping metadata, custom S3 classes,
attributes, row order, factor levels, and zero-row behavior. Reconstruction may
drop or restore attributes differently from a vectorized replacement, so values
can match while complete objects differ. If compatibility requires the previous
reconstruction behavior, reproduce it deliberately rather than assuming which
attributes should survive.

Compare full tibbles and nested result objects against an installed baseline
(`waldo::compare()` component by component); value-level comparison misses
this class of difference. See
[benchmarking-validation.md](benchmarking-validation.md).

Do not substitute base R or `data.table` throughout merely because a synthetic
microbenchmark is faster. A small private base-R helper is reasonable when it
preserves the package's idiom at the boundary, is clearly documented, and has
a material measured benefit — preserve the public interface and optimize
below it. Accelerator backends (`dtplyr`, `collapse`, `duckplyr`) are
dependency-posture decisions to flag with evidence, not to impose.

## Budget

Tidy packages are often exploratory tools run a handful of times per analysis,
not inner loops. Milliseconds of labeled-output bookkeeping in exchange for
never debugging a row-ordering mismatch is a good trade; state that trade-off
in the performance docs and set an explicit revisit threshold (e.g. revisit if
the gap to a matrix-native alternative exceeds an order of magnitude on
realistic inputs) instead of chasing parity.
