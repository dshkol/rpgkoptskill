---
id: 2026-08-16-vincentarelbundock-wdi-pagination-metadata
package: WDI
repository: https://github.com/vincentarelbundock/WDI
status: sent
drafted: 2026-08-16
sent: 2026-08-16
last_checked: 2026-08-16
external_link: https://github.com/vincentarelbundock/WDI/pull/71
proposed_commit: https://github.com/vincentarelbundock/WDI/pull/71/commits/c519c693674c1d3f3d3addc1115bae11fbcefda0
---

# Stop requests after the API-reported final page

Optimize `wdi.dl()` by stopping its request loop after receiving the final page
reported by the World Bank API. Preserve the existing bounded fallback when
page-count metadata is absent, malformed, or unusable.

## Investigation context

- Versions or commits: proposed in
  [PR #71](https://github.com/vincentarelbundock/WDI/pull/71) at
  [`c519c69`](https://github.com/vincentarelbundock/WDI/pull/71/commits/c519c693674c1d3f3d3addc1115bae11fbcefda0),
  targeting base commit `22ed43e`.
- Workload and scale: public WDI data downloads whose API responses contain one,
  five, or nine pages; the existing implementation constructs and may request
  ten page URLs per indicator.
- Compatibility boundary: no public API change; complete returned objects must
  remain identical; the ten-page upper bound remains; unsuccessful or empty
  responses still terminate pagination; missing, non-numeric, non-finite,
  fractional, or otherwise invalid page metadata uses defensive fallback.
- Environment and network policy: deterministic mocked requests with 10 ms
  simulated delay and 20 repetitions after warm-up; two uncontrolled live API
  observations reported separately.

## Evidence

Deterministic request-count results:

| Reported pages | Baseline requests | Candidate requests |
|---:|---:|---:|
| 1 | 10 | 1 |
| 5 | 10 | 5 |
| 9 | 10 | 9 |

- **Demonstrated:** candidate request count equals the valid API-reported page
  count rather than the fixed maximum. Complete outputs were identical for all
  three fixtures.
- **Modeled:** with a fixed 10 ms delay per request, elapsed differences model
  request-bound work rather than World Bank server latency.
- **Illustrative live observations:** United States for 2000-2020 fell from
  5.906 seconds to 0.208 seconds, and all countries for 1960-2025 fell from
  9.532 seconds to 0.398 seconds. Both returned identical objects and reduced
  requests from ten to one, but each timing was a single uncontrolled network
  observation.
- **Validation:** 13 pagination-specific tests passed; the full suite reported
  54 passed, no failures, warnings, or skips; a fresh source-package check
  reported `Status: OK`.
- **Durable artifacts:** implementation and regression fixtures are in the PR.
  The extensive audit and benchmark harness were intentionally kept outside the
  target package, and no portfolio copy is currently available.

## Recommendation

Read and validate the response's `pages` value, then stop after the reported
final page. Inject the JSON reader only at the internal boundary needed for
deterministic testing. Remove pagination metadata from assembled data so public
outputs remain unchanged. Retain empty/error termination and the existing
ten-page cap whenever the metadata cannot be trusted.

Accept only with request instrumentation, complete output-object equivalence,
normal and malformed metadata coverage, preserved failure behavior, full tests,
and package check.

## Outcome

Current outcome: unknown

The recommendation is public in an open, unmerged PR. As of the last check it
had no maintainer comments or reviews, so no acceptance or rejection is
inferred.

| Date | Status | Evidence and reason |
|---|---|---|
| 2026-08-16 | draft | Recommendation prepared from the pagination audit. |
| 2026-08-16 | sent | Opened [PR #71](https://github.com/vincentarelbundock/WDI/pull/71). |

## Learnings

### Package-specific

- `wdi.dl()` may request a fixed ten pages per indicator even when the first API
  response reports that fewer pages exist. Remote request orchestration, not a
  local R micro-operation, was the actionable cost.
- Even all countries across 1960-2025 fit in one response for the observed live
  query. Large-looking user inputs do not necessarily imply many API pages when
  the service uses generous page sizes.
- The package's existing ten-page construction, empty-response termination, and
  error handling provide a defensive fallback that can coexist with the normal
  metadata-aware path.

### Transferable

- Audit pagination metadata, fixed page caps, cursors, and termination logic
  before pursuing local micro-optimizations in API clients.
- Treat request count as the primary reproducible metric. Keep network timings
  secondary and label uncontrolled measurements as illustrative.
- Validate impact against real response shapes before calling an optimization
  high impact.
- Benchmark through the public workflow with isolated baseline and candidate
  installations, mocking only the remote boundary and comparing complete
  outputs.
- Preserve defensive behavior when pagination metadata is missing, malformed,
  inconsistent, or unavailable.
- Separate demonstrated resource reduction, modeled elapsed benefit,
  uncontrolled live observation, and likely real-world impact.
- Keep a focused upstream PR small while retaining extensive audit and
  benchmark artifacts in the portfolio when possible.

Valid when an API client speculatively requests a fixed page range and a
response provides trustworthy completion metadata or cursors.

### Negative results

1. **Assume a broad query must span many pages:** all countries across 65 years
   still fit in one observed World Bank response. Query dimensions alone were
   not a reliable proxy for request count.
2. **Lead with live timing multipliers:** live checks showed large latency
   differences but were uncontrolled single observations. They support
   plausibility, not a reproducible speedup claim.
3. **Optimize local R processing first:** avoidable remote calls dominated the
   mechanism, so local parsing or allocation tuning would have targeted a
   secondary cost.
4. **Commit the full performance audit upstream:** unnecessary benchmark
   infrastructure would enlarge a focused maintainer-facing change without
   improving the permanent regression tests.

## Follow-up

- Action: update status and timeline when maintainers respond, the PR changes,
  merges, closes, or ships; preserve the benchmark harness in the portfolio if
  it becomes available.
- Owner: portfolio maintainer
