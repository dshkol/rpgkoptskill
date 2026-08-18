# data.table packages

Preserve `data.table`'s reference semantics and established syntax. Mutation,
keys, secondary indices, automatic optimization, and threads affect both
behavior and benchmark validity.

## Optimize within the model

- Use keyed joins, secondary indices, and `on=` joins when repeated lookup or
  merge patterns justify them. Measure index construction separately from
  indexed queries.
- Match the join primitive to the relationship: non-equi `on=` joins for
  inequalities, `roll=` for ordered nearest or last/next-observation matches,
  and `foverlaps()` for intervals. When the result is an aggregate per row of
  `i`, use `by = .EACHI` rather than materializing a join and grouping it again.
- Keep `j` expressions in shapes GForce can optimize (`.N`, `mean`, `sum`,
  `min`, `max`, `first`, `last`, and friends applied directly to columns).
  Optimization coverage changes across `data.table` versions, so do not
  fossilize the current expression whitelist: use `verbose = TRUE` to confirm
  the hot query is optimized on the package's minimum and tested versions.
- When only a filtered row count is needed, prefer `DT[i, .N]` to
  `nrow(DT[i])`; the latter materializes the subset merely to count it.
- Use `rowid()` for occurrence numbers within groups and `rleid()` for IDs of
  consecutive runs instead of constructing those groupings manually.
- Use `:=` for ordinary by-reference column updates. Inside a proven hot loop,
  consider `set()` to avoid repeated `[.data.table` dispatch.
- Inside hot loops, consider `setDT()` or `as.data.table()` on valid lists rather
  than repeatedly constructing with `data.table()`; use `setnames()`,
  `setorder()`, and `setcolorder()` over copying equivalents.
- Prefer `%chin%` over `%in%` for character membership, `fifelse()`/`fcase()`
  over `ifelse()` chains, and `uniqueN()` over `length(unique())` in hot
  paths.
- Avoid accidental materialization and `copy()`. Copy only at an ownership
  boundary that requires isolation.
- Batch groups and joins where repeated setup dominates, but do not replace an
  idiomatic fast grouped expression without evidence.

## API-boundary ownership

Inside package internals, by-reference mutation is the idiom. At the exported
boundary it is a contract decision: a user-facing function that mutates its
input (adds columns, sets keys, reorders) is observable behavior. Either
document reference semantics explicitly or take one deliberate `copy()` at
entry — and state which, so later optimization does not flip it silently.
Check whether callers observe mutation, keys, index attributes, column order,
names, and object identity. A candidate using `copy()` may be semantically
safer but erase the measured gain; a candidate mutating the input may be
faster but wrong.

## Threads

`data.table` is multithreaded by default but CRAN checks run with two threads,
and user machines vary. Record `getDTthreads()` in benchmarks, expose thread
control rather than calling `setDTthreads()` on load with a value the user did
not choose, and never report a speedup that is actually a thread-count
difference.

## Benchmark correctly

- Each by-reference iteration must start from an equivalent fresh object.
  Timing repeated mutation of one object measures different work.
- Separate cold index construction, warm indexed lookup, and end-to-end user
  cost. Automatic indices can make later iterations unrepresentative.
- For `fread()`, distinguish disk/OS cache effects from parsing. Also measure the
  downstream pipeline because R's character-string cache can shift costs.
- Avoid class coercion in a benchmark unless it is part of the real workflow.
