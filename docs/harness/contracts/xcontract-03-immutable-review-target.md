# XCONTRACT-03 — Immutable Review Target Semantics

- status: **Committed Contract Baseline**
- promoted_at: 2026-08-28
- promotion_issue: #5
- review_transport: PR #4
- normative_candidate_file: `review-candidates/2026-08-28-xcontract-03-immutable-review-target-v5.md`
- normative_candidate_commit: `39c7a9b755567792e343bec94a7dd8725c5d37a5`

## Normative definition

The normative XCONTRACT-03 contract semantics are **exactly** the reviewed Design Candidate v5 at commit:

`39c7a9b755567792e343bec94a7dd8725c5d37a5`

This baseline record promotes that immutable revision **by reference**. It does not rewrite, normalize, restate, or silently synthesize a v6.

Historical Candidate revisions v1-v4 and their reviews remain provenance only and are not promotion targets.

## Committed invariants

The promotion accepts the following invariants from the exact v5 revision:

1. A Review Target binds one exact Run-scoped State Revision and one exact immutable Candidate Revision; it never points to a live `current` / `latest` Candidate.
2. Candidate finalization owns target-semantic closure. Every correctness-relevant dependency must be frozen/materialized and deterministically recoverable before the Candidate Revision becomes reviewable.
3. Review Snapshot capture begins only after Candidate finalization and is limited to **select / validate / persist binding**.
4. Snapshot capture cannot freeze missing Candidate semantics, mutate the selected Candidate Revision, silently substitute another revision, or attach Snapshot-only semantic overrides.
5. If the selected Candidate Revision does not already have a frozen/recoverable target-semantic closure, Snapshot capture fails with `CANDIDATE_NOT_REVIEWABLE` (or an equivalent explicit reviewability failure).
6. Candidate revision identity answers **which immutable revision** is targeted. Canonical payload/hash verifies integrity and content equivalence. Hash equality does not replace revision identity.
7. State Revision identity is Run-scoped: `(run_id, state_version)`.
8. Review Snapshot identity and reviewer-execution provenance are distinct concerns.
9. `stale` / freshness is a factual relation used for revalidation; it is not automatically a Review Issue disposition.
10. This contract introduces no new durable Runtime Object and grants no new canonical mutation authority to Design, Review, or Integration roles.

## Explicitly rejected semantics

The committed baseline rejects:

- live `current candidate` / `latest candidate` references as Review Targets;
- Snapshot-time mutation or silent replacement of Candidate Revision;
- Snapshot-only semantic overrides used to compensate for incomplete Candidate semantics;
- treating content hash alone as Candidate revision identity;
- allowing Snapshot capture to perform Candidate finalization.

## Scope boundary

This promotion closes **XCONTRACT-03 only**.

It does not promote or close Continuation rollover bootstrap, Continuation operation idempotency, Execution Journal / Recovery, or other Cross-Contract issues tracked separately.

For any ambiguity, the exact v5 candidate at the normative commit above is authoritative over this promotion record's explanatory summary.
