# Geospatial packages

Spatial cost is often controlled by extent, resolution, vertex count, CRS
transformation, topology, or candidate-pair count rather than row count alone.
Record these dimensions in every fixture.

## High-value opportunities

- **Look for formulations whose cost tracks the answer, not the extent.** For
  example, a minimal grid covering of points need not generate every candidate
  cell across the bounding box and filter by intersection. When the contract
  permits, directly encoding the points and taking unique cell identifiers can
  make work track point count instead of area and resolution. Benchmark across
  extent and precision before adopting the fast path.
- Use bounding boxes, spatial indices, prepared geometries, or other necessary
  conditions to prune candidates before exact predicates or distances. `sf`
  binary predicates commonly build an index on the first argument; consult the
  predicate documentation and benchmark argument order for the actual geometry
  sizes and backend. Pre-filter only with bounds that preserve the predicate's
  semantics.
- Avoid encode/decode and class-conversion round-trips. Convenience layering
  can create paths such as
  `grid → encode → class conversions → decode → polygons`, encoding
  coordinates only to immediately decode them. Test direct construction from
  the earliest suitable representation. Internal reuse of
  user-facing convenience constructors is a smell in hot paths — they exist
  for API ergonomics, not throughput. Same family: repeated
  `sp`/`sf`/GeoJSON/WKT conversions and re-extracting `st_coordinates()` per
  step.
- Transform CRS once at a clear boundary. Do not repeat transformations per
  feature or inside nested helpers. The same applies to validity checks
  (`st_make_valid`, `st_is_valid`) — establish validity once and record it.
- Use vectorized geometry operations or matrix-aware spatial-lag operations
  instead of one call per feature or time period — `spdep::lag.listw()`
  accepts a units-by-periods matrix and lags every column in one call.
- Simplify, chunk, tile, or stream only under an explicit accuracy and output
  contract. Never treat geometry simplification as behavior-preserving by
  default.
- Enforce size limits after expansion into the representation actually parsed
  or embedded; compressed source size does not bound GeoJSON or R-object size.

## Benchmark shape

Vary feature count, vertex count, spatial spread, resolution/precision, overlap,
CRS, geometry validity, and candidate count. Report the intermediate driver,
such as grid cells or predicate candidates, alongside time and memory — the
mechanism column is what makes a spatial scaling claim auditable. Mark
baseline cases that cannot run at all as *impractical* rather than omitting
them.

## Semantic traps

Compare CRS, precision, geometry type, empty geometries, validity, dimensionality,
feature order, attribute-geometry relationships (`agr`), boundary predicates,
and backend choice (`s2` versus planar operations). Test antimeridian, poles,
empty and invalid geometries, and units where relevant.

## Substrate churn

Before optimizing a representation or dependency, check whether the ecosystem
is migrating away from it — `sp` → `sf` and `raster` → `terra` both stranded
finished optimization PRs when maintainers rewrote the same functions on the
successor. The algorithmic insights (answer-scaled fast paths, direct
construction, single-pass dedup) survive a port; patches against the retiring
substrate do not. Optimize the successor layer or the
representation-independent layer, land small PRs quickly, and write up
transferable findings separately from the patches.
