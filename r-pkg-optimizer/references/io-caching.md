# Remote APIs, files, databases, and caches

Separate network latency, request count, server work, disk access, decompression,
parsing, deserialization, validation, and downstream transformation. Optimize
the stage users actually wait for.

## I/O and batching

- Batch requests or queries when supported, while respecting rate limits,
  maximum payloads, retries, partial failure, and ordering.
- Select and filter at the authoritative backend when it reduces transfer and
  preserves semantics; avoid repeated round-trips for small pieces.
- Vectorize file metadata and parsing calls over paths rather than creating a
  one-row result per file — one `file.info(paths)` call replaces a loop of
  per-file calls.
- Stream or chunk inputs when peak memory is the constraint, but include parsing
  overhead and output ordering in the contract.
- Reuse connections and prepared state only where lifecycle and fork safety are
  understood.
- For metadata-only reads of remote structured files, test storage- and
  format-native random access before downloading complete objects or
  implementing range parsing. Instrument bytes or requests to verify that the
  apparent random-access API does not silently materialize the full object,
  and retain a portable fallback when support is optional.
- For paginated APIs, inspect `pages`, `total`, cursors, and completion tokens
  before optimizing local R work. Instrument request count, test real one- and
  multi-page response shapes plus missing or malformed metadata, and retain a
  bounded fallback when termination metadata cannot be trusted.

## Layered caching

A useful hierarchy is session memory, then durable local cache, then remote
source. **A disk cache is not necessarily a fast in-session cache**: every hit
still pays read, decompression, and unserialization. When the call graph shows
the same immutable object loaded repeatedly, benchmark a session environment
in front of the file cache:

```r
# Package-level session cache fronting the existing file cache
the_cache <- new.env(parent = emptyenv())

get_metadata <- function(key, use_cache = TRUE) {
  if (use_cache && !is.null(v <- get0(key, envir = the_cache))) return(v)
  v <- read_file_cache_or_download(key, use_cache = use_cache)
  assign(key, v, envir = the_cache)   # refresh reaches this layer too
  v
}
```

Before adding or changing a cache, define:

- key completeness and canonicalization;
- value size and memory/disk budget;
- freshness, TTL, versioning, and schema migration;
- explicit refresh and invalidation across every layer;
- concurrent readers/writers and atomic writes;
- failure, corruption, permissions, privacy, and credential handling;
- session, process, user, and platform scope.

Route existing `use_cache = FALSE`, refresh, or force flags through **every**
cache layer — a new memory layer that ignores a documented refresh flag
violates the refresh contract even though every individual read is correct.
Avoid caching cheap work or high-cardinality results without evidence.

For placement: `tempdir()` for session-scoped caches,
`tools::R_user_dir(pkg, "cache")` for persistent ones — CRAN policy forbids
writing elsewhere in the user's filesystem by default.

## Benchmarking

Use realistic offline fixtures and mocked bindings for API packages — see the
recipe in [benchmarking-validation.md](benchmarking-validation.md). Measure
cold remote-equivalent, warm disk, and warm memory paths separately. Count
requests and bytes as well as elapsed time. Do not publish speedups dominated by
an unreported pre-warmed OS or application cache.
