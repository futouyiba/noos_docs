# Independent Reviewer Packet — Deliberation Harness v4

## Review Target

Review **NOOS Deliberation Harness — Review Candidate v4** from GitBook frozen revision `gXJhYz6y0hxslcRqGgcE`.

This is a narrow re-review after v3. Do not redesign the whole Harness unless a concrete counterexample requires it.

## What v4 intentionally changed

Review whether these closures are sufficient and minimal:

1. unresolved current-slice `Needs human/evidence` is blocking unless explicitly moved to `Defer with trigger`;
2. Proposed Diff explicitly declares `incorporated_inbox_item_ids` distinct from selected items;
3. stale-after-switch is derived context mismatch, not durable state;
4. `selected` is temporary Absorb context, not Inbox state;
5. Inbox durable states are reduced to pending / absorbed / closed(reason);
6. Freeze Snapshot includes Work Item identity and fixed supporting-reference identities;
7. cold-start has a fixed rubric and remains evidence only;
8. Review Note uses `addressed-by-designer` to preserve Reviewer authority boundary;
9. Promote is a lightweight Human authority event referencing immutable records rather than copying them.

## Primary attack questions

- Can a current-slice product decision still bypass `blocking: yes` and be Promoted?
- Can Inbox absorbed state still differ depending on implementation inference?
- Can cross-Work-Item switching or base revision drift still cause silent mutation?
- Is Freeze Snapshot reproducible enough without becoming a large duplicated artifact?
- Does Human Promote now have enough authority semantics without becoming another document workflow?
- Has v4 accidentally reintroduced durable ceremony via `blocking`, `closed(reason)`, Snapshot, or Promote?
- Can another 25–50% of user-facing workflow still be removed without reopening correctness gaps?

## Non-goals

Do not require:

- Section-local staging;
- durable Review Staging;
- normative inline marker syntax;
- full Review Issue tree / lineage;
- multi-agent review orchestration;
- a second/third readiness gate;
- Codex runtime control.

## Desired verdict

State explicitly whether v4 is:

1. small enough for the first self-bootstrapping implementation slice;
2. semantically closed enough to enter final Human `implementation-safe` Promote;
3. still missing any product decision that would force Codex to invent observable product behavior, authority, or failure semantics.

For each blocking/major issue, provide one concrete counterexample and the narrowest fix possible.
