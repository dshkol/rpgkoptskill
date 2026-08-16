---
id: 2026-08-09-noahdasanaike-sage-random-access-schema
package: sage
repository: https://github.com/noahdasanaike/sage
status: implemented
drafted: 2026-08-09
sent: 2026-08-09
last_checked: 2026-08-16
external_link: https://github.com/noahdasanaike/sage/pull/118
implemented_commit: https://github.com/noahdasanaike/sage/commit/aab19c750e94b7d4170df3e71467dfe16665e45c
---

# Read GCS-backed Parquet schemas through Arrow random access

Optimize cold `sage_columns()` calls by reading Parquet schema metadata through
Arrow's anonymous GCS random-access filesystem when the source is GCS-backed.
Preserve the existing full-download path for unsupported Arrow builds, non-GCS
sources, or failed random-access reads.

## Investigation context

- Versions or commits: proposed and shipped in
  [PR #118](https://github.com/noahdasanaike/sage/pull/118), implemented by
  [`aab19c7`](https://github.com/noahdasanaike/sage/commit/aab19c750e94b7d4170df3e71467dfe16665e45c).
- Workload and scale: a cold `sage_columns()` call over the six representative
  production Parquet partitions; the baseline downloaded about 298.3 MB only
  to inspect schemas.
- Compatibility boundary: no public API change; identical ordered result; the
  same six-partition selection and schema-union behavior; existing session
  cache preserved; no new dependency; both `gs://` and
  `storage.googleapis.com` source forms supported; portable fallback retained;
  APIs compatible with the declared Arrow 14 minimum.
- Environment and cache/thread policy: matched live-archive cold calls with
  three randomized iterations per implementation and the session cache cleared;
  warm session-cache behavior measured separately.

## Evidence

Matched cold-call results:

| Implementation | Median | Range |
|---|---:|---:|
| Full downloads | 5.87 s | 5.80-6.00 s |
| Random-access GCS | 2.63 s | 2.55-2.79 s |

- **Demonstrated:** 2.24x faster at the median, saving 3.24 seconds, or about
  55%. Warm cached calls remained about 4.4 microseconds.
- **Observed:** the full-download baseline transferred about 298.3 MB to answer
  a metadata-only query.
- **Validation:** `identical()` covered the exact ordered 35-column public
  result; nine real individual Parquet schemas; all six production partitions;
  a 17 KB partition; a percent-encoded country path; a materially different
  German schema; and complete `sage_load()` outputs for Samoa, Bosnia and
  Herzegovina, and a 1,073,412-row filtered German result.
- **Compatibility checks:** explicit HTTPS GCS input, non-GCS and failure
  fallbacks, Arrow 14 APIs, and package build/check with no new findings.
- **Durable artifacts:** the PR records results and environment details, but the
  benchmark harness was not preserved as a reusable repository artifact.

## Recommendation

Use Arrow's storage-native anonymous GCS random access for the metadata-only
path. Detect support at runtime and retain the existing full-download behavior
as a trustworthy fallback. Accept only with exact schema/result equivalence,
unchanged partition selection and caching semantics, Arrow 14 compatibility,
and a demonstrated improvement in the cold public workflow.

## Outcome

Current outcome: implemented

The exact proposed commit shipped upstream. This is direct implementation
evidence, not acceptance inferred from discussion or an open PR.

| Date | Status | Evidence and reason |
|---|---|---|
| 2026-08-09 | draft | Recommendation prepared from the measured cold workflow. |
| 2026-08-09 | sent | Opened [PR #118](https://github.com/noahdasanaike/sage/pull/118). |
| 2026-08-16 | implemented | Verified upstream commit [`aab19c7`](https://github.com/noahdasanaike/sage/commit/aab19c750e94b7d4170df3e71467dfe16665e45c). |

## Learnings

### Package-specific

- `sage_columns()` is a metadata query implemented over very large data
  objects. Its cost scaled with sampled Parquet file size while its answer
  scaled only with schema metadata. The archive's GCS location let Arrow resolve
  that mismatch without changing the archive or package API.
- A narrow performance contribution was a better first contribution than a
  broad package-hygiene rewrite. It respected the existing design and delivered
  immediate user value without imposing a roadmap on the maintainer.

### Transferable

- Look for metadata-only operations that materialize entire remote objects.
- Before implementing protocol mechanics, test whether the existing backend
  exposes genuine storage- and format-native random access. Verify that the API
  does not silently materialize the complete object.
- Use discarded prototypes to test mechanism viability and estimate a ceiling,
  while applying the complexity veto to production candidates.
- Measure transfer and latency separately. The footer prototype reduced bytes
  about 34x; the shipped implementation reduced median latency 2.24x. They are
  different claims.
- Preserve capability-dependent optimizations with runtime detection and a
  trustworthy fallback when optional features vary across dependency builds.
- Treat contribution scope as part of optimization quality. A small production
  diff can require broad validation, and the strongest contribution is not
  necessarily the largest available cleanup.

Valid when a workflow needs only metadata or a small region of a remote object
and the storage/format library supports genuine random access.

### Negative results

1. **Custom HTTP-range Parquet footer parser:** transferred 8.78 MB instead of
   298.29 MB and matched three real schemas, but required roughly 40-50 lines of
   bespoke footer and range-handling logic. Arrow's GCS reader achieved the
   user-visible goal with much less maintenance surface.
2. **Open HTTPS URLs directly with `ParquetFileReader`:** appeared simpler, but
   inspection showed Arrow downloaded the complete file before reading the
   schema. Syntactic directness did not reduce measured work.
3. **Published schema sidecar:** architecturally clean, but required a change to
   the upstream data-production pipeline and was unnecessary after proving the
   client-side storage-native path.
4. **Broad package-health PR:** several legitimate check issues were found but
   deliberately excluded because they would obscure the focused performance
   change and presume the maintainer's packaging roadmap.

## Follow-up

- Action: preserve reusable benchmark harnesses in this portfolio for future
  recommendations when adding them to the target package would enlarge a
  focused PR unnecessarily.
- Owner: portfolio maintainer
