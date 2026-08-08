# HTML widgets, JavaScript, and serialization

Profile the full R-to-browser path: R-object construction, validation or
redaction, JSON serialization, HTML embedding, transfer, browser parsing,
JavaScript initialization, rendering, and interaction. The dominant boundary
may not be where R profiling points first.

Measure stages separately before choosing a target. A nested R representation
can be much larger than its compact JSON output, and serialization can dominate
an otherwise fast construction and transformation pipeline. Do not optimize an
already-cheap helper while users wait at the representation boundary.

## R representation and JSON

- Measure R object size, serialized bytes, serialization time, and peak memory
  per stage. Nested inspectable lists may be an order of magnitude larger than
  their compact JSON output.
- Construct final feature or record lists directly from columns; avoid one-row
  data frames and repeated list traversal in measured hot paths. See
  [generic-r.md](generic-r.md).
- Remove representation round-trips and repeated serialization. Cache only with
  correct invalidation and an acceptable memory budget.
- Do not infer that a wrapper such as `gsub()` is expensive merely because it
  is the outer call in a profile — check self time; its argument evaluation
  may contain the actual serializer cost.
- Benchmark serializers against exact JSON shape, numeric precision, missing
  values, escaping, key order if contractual, and JavaScript consumer behavior
  before adding a dependency. A faster serializer is a dependency-posture
  decision to flag, not impose.
- Pretty output often changes bytes more than CPU. Describe compact output as a
  size/transfer option, not a speed optimization, unless measurement shows
  otherwise for the package's payloads.

## Large payload strategies

When embedding dominates, consider chunking, streaming, external data files,
lazy/browser-side loading, aggregation, or an explicit size threshold. These
can change portability, offline behavior, security, deployment, and API
semantics; obtain direction for user-visible changes.

Apply limits to expanded GeoJSON/JSON or the browser-bound object, not only
the compressed input file — compressed vector files, `sf` objects, and
converted GeoJSON expand dramatically, and a source-file limit does not bound
the expanded object. Fail before the final expensive parse and copy.

## JavaScript side

- Profile browser parse, scripting, layout, paint, memory, and interaction on a
  realistic browser/device before changing bundling or rendering strategy.
- Batch DOM or map-layer updates; avoid repeated layout-triggering reads and
  writes; virtualize or canvas-render large collections when consistent with
  the widget's design.
- Move computation browser-side only when transfer reduction exceeds browser
  cost and the result remains deterministic and testable.
- Keep generated bundle changes separate from source changes and run the
  package's build plus browser/widget tests.

Validate standalone HTML, dependency resolution, CSP/security behavior,
headless rendering, resize, incremental updates, and browser-visible equality.
