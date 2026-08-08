# R Package Optimizer Skill

A reusable agent skill for profiling, optimizing, and validating performance in
existing R packages — while preserving each package's public API, semantics,
dependency posture, and chosen implementation style. Companion to
[rpkgdevskill](../rpkgdevskill), which covers general R package development.

## What This Skill Does

When activated, this skill helps an agent with:

- Profiling realistic public workflows and attributing cost to the right stage
- Designing honest benchmarks (scaling tables, offline API mocking, installed
  baselines) and reporting evidence proportional to the claim
- Applying context-appropriate optimizations: generic/base R, tidyverse,
  data.table, C/C++/Rcpp, Rust, geospatial, HTML widget/JSON, API/caching, and
  numerical/parallel workloads
- Verifying behavior-preserving refactors with full-object equivalence
  harnesses against installed baselines
- Deciding when a native backend is justified — and flagging it with evidence
  rather than implementing it uninvited

## Design principles

- **Respect the package's design context.** A tidyverse package stays
  tidyverse; data.table stays data.table. The goal is the fastest version of
  the package it already is, not a different package.
- **Generic improvements are always fair game**: dead work, better algorithms,
  vectorization, caching, allocation reduction.
- **Native backends are an escalation, never a default.** Optimizing an
  existing C/C++/Rust backend is in scope; adding one to an R-only package is
  flagged with profiling evidence and left to the user's direction.
- **Measured, verified, honest.** Profile before believing any hypothesis;
  compare complete objects against an installed baseline; report scaling
  behavior, not one-machine multipliers.

## Installation

Copy the skill directory to your Claude Code skills location:

```bash
# For personal use (available across all projects)
cp -r r-pkg-optimizer ~/.claude/skills/

# Or for a specific project
cp -r r-pkg-optimizer /path/to/your/project/.claude/skills/
```

Then restart Claude Code to load the skill.

For Codex, copy the same directory into `$CODEX_HOME/skills/` (or
`~/.codex/skills/` when `CODEX_HOME` is unset). The optional
`agents/openai.yaml` supplies Codex-facing display metadata.

## Skill Contents

### Skill Directory (`r-pkg-optimizer/`)
- `SKILL.md` — governing principles, core workflow, architecture routing, and
  the native-backend escalation policy

### Reference Documentation (`r-pkg-optimizer/references/`)
- `generic-r.md` — idiom-neutral patterns: estimation anchors, dead work,
  scaling, allocation, fast paths
- `profiling-feedback.md` — profiling commands, evidence capture, and
  review/issue/PR formats
- `benchmarking-validation.md` — benchmark contracts, fixtures, equivalence
  harnesses, offline API benchmarks, evidence reporting
- `tidyverse.md` — grouped-dispatch hot spots, boundary-masked vectorization,
  semantic traps
- `data-table.md` — reference semantics, GForce, keys, threads, API-boundary
  ownership
- `native-cpp.md` — existing C/C++/Rcpp/cpp11 backends: boundary costs, hot
  loops, validation
- `rust.md` — existing Rust/extendr backends
- `geospatial.md` — answer-scaled formulations, pruning, round-trips,
  substrate churn
- `web-serialization.md` — HTML widgets, JSON, and browser boundaries
- `io-caching.md` — APIs, files, layered caches, invalidation contracts
- `numerical-parallel.md` — matrix/statistical workloads and the parallelism
  gate
- `sources.md` — authoritative external references and further reading

## Usage

The skill triggers on R package performance tasks such as:

- "This function/package is slow" or a reported performance regression
- Profiling, benchmarking, or `Rprof`/`profvis`/`bench` work
- Vectorization, allocation, memory, or serialization problems
- Reviewing or writing performance PRs and issues
- Deciding whether C/C++/Rust is warranted

## License

MIT
