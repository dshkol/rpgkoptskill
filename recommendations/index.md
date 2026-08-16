# External package recommendations

This ledger tracks performance recommendations, their externally verified
outcomes, and lessons from package investigations. Records live in
[`records/`](records/), with reusable benchmark materials under `artifacts/`
when available, and follow the protocol in
[`recommendation-history.md`](../r-pkg-optimizer/references/recommendation-history.md).

| ID | Package | Recommendation | Status | External link | Last checked |
|---|---|---|---|---|---|
| [2026-08-09-noahdasanaike-sage-random-access-schema](records/2026-08-09-noahdasanaike-sage-random-access-schema.md) | sage | Read GCS-backed Parquet schemas through Arrow random access | implemented | [PR #118](https://github.com/noahdasanaike/sage/pull/118) | 2026-08-16 |
| [2026-08-16-vincentarelbundock-wdi-pagination-metadata](records/2026-08-16-vincentarelbundock-wdi-pagination-metadata.md) | WDI | Stop requests after the API-reported final page | sent | [PR #71](https://github.com/vincentarelbundock/WDI/pull/71) | 2026-08-16 |

Add one row when creating a record. Keep the status and last-checked date in
sync with the record; use `unknown` rather than guessing an outcome.
