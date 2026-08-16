# Recommendation history

Keep a portfolio-level record of performance recommendations made to packages
outside the portfolio repository. Treat the record as an evidence trail and a
learning system, not as a scorecard for maintainers.

## Choose the record boundary

Use one record per independently actionable recommendation. Keep related
benchmark variants and follow-up discussion in that record. Create separate
records when maintainers could accept one proposal without accepting the
other.

Search the portfolio ledger by repository, issue or PR URL, affected workflow,
and recommendation summary before assigning a new ID. Update an existing
record when new measurements, maintainer feedback, releases, or reversions
change the outcome.

Do not create a record for a hypothesis that was dismissed before it became an
actionable recommendation. Preserve that negative result in the parent
investigation or in the learning section of a related record.

## Record the recommendation

Use a stable ID of the form `YYYY-MM-DD-owner-repository-slug` and record:

- package and repository;
- affected public workflow and versions or commits investigated;
- exact recommendation and its compatibility boundary;
- observed evidence, reproduction command, and durable artifact links;
- external issue, review, PR, or discussion link, if sent;
- status and dates first drafted, sent, and last checked;
- outcome evidence and the reason accepted, changed, declined, or withdrawn;
- package-specific findings, transferable lessons, and negative results;
- follow-up action and owner, if any.

Use only these lifecycle states so the ledger remains searchable:

- `draft`: actionable, but not communicated externally;
- `sent`: communicated; no disposition is known;
- `accepted`: maintainers agreed, but implementation is not yet verified;
- `partially-accepted`: only a documented part was accepted;
- `implemented`: verified in the default branch or a release;
- `declined`: maintainers explicitly chose not to proceed;
- `withdrawn`: evidence or constraints no longer support the recommendation;
- `superseded`: another change or recommendation replaced it.

Never infer acceptance from silence, a reaction, or an open PR. Link the event
that supports each status transition and retain the previous status in the
timeline.

## Write useful learnings

Separate three kinds of learning:

1. **Package-specific:** architecture, contracts, workload shapes, maintainer
   preferences, and constraints needed for future work on that package.
2. **Transferable:** a mechanism, benchmark design, semantic trap, or review
   pattern worth applying elsewhere, with the boundary where it is valid.
3. **Negative result:** an attractive idea that failed measurement, semantics,
   or maintenance-cost review, including enough context to avoid repeating it.

State observations separately from interpretations. Do not generalize from one
package without naming the conditions that made the lesson true.

## Keep records safe and current

- Store the log in an explicitly designated portfolio repository, separate
  from the external package under review. In this skill's source repository,
  use `recommendations/index.md`, `recommendations/records/`, and
  `recommendations/artifacts/<recommendation-id>/`.
- Link reproducible public artifacts when possible. Do not commit credentials,
  private correspondence, proprietary fixtures, or machine-specific paths.
- Record `unknown` when an outcome has not been checked; do not convert missing
  evidence into a conclusion.
- Update the record after material external events. Do not poll maintainers or
  post follow-ups unless the user authorized that interaction.
- If the ledger is unavailable or read-only, emit a complete Markdown record
  for the user to file later.
