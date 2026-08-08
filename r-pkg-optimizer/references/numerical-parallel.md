# Numerical, statistical, and parallel workloads

## Numerical and matrix work

- Batch scalar operations into vector, matrix, sparse-matrix, or factorization
  routines when the structural contract permits it.
- Reuse decompositions, sufficient statistics, design matrices, and index maps
  across repeated calls when inputs are unchanged.
- Use symmetry, sparsity, banding, monotonicity, and problem-specific structure
  before tuning arithmetic.
- Avoid materializing large dense pairwise matrices when only neighbors,
  aggregates, or thresholded results are needed.
- Match precision and storage to the documented method; reducing precision is
  not behavior-preserving by default.
- For already-aggregated small matrices, measure once and usually stop. An
  `O(k^3)` method at `k <= 10` is often irrelevant to end-to-end cost.

Validate convergence, conditioning, seeds, tolerances, attributes, and edge
cases with an independent oracle or mathematical invariants. A faster method
that converges differently is a methodological change requiring explicit
review.

## Parallelism gate

Use parallelism only after reducing total work and only for a measured,
independent, coarse-grained workload. Estimate whether task size repays startup,
serialization, scheduling, synchronization, and result combination.

Before implementation, define:

- supported operating systems and fork/socket constraints;
- worker and BLAS/`data.table`/OpenMP thread interaction;
- deterministic RNG streams and output ordering;
- error, warning, interrupt, and cleanup behavior;
- memory multiplication and data export to workers;
- user control and a serial fallback.

Avoid nested or implicit oversubscription. Benchmark throughput and peak memory
at realistic core counts, including one core. Parallelism can reduce latency
while consuming more total CPU and memory; report both when relevant.
