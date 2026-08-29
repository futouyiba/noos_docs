# Independent Reviewer Packet — Deliberation Harness v3

> GitBook frozen revision: `fggdIa5FgR5Xcj6N9Zg9`

## Review assignment

Review only **NOOS Deliberation Harness — Review Candidate v3** from the same GitBook revision as this packet.

This is a narrow re-review after v2. Do not redesign the whole Harness unless a concrete counterexample proves the simplified v0 still fails.

## What v3 intentionally changed

v3 only tightens:

1. creation-time Work Item ownership and stale-after-switch semantics;
2. minimum Inbox retrieval behavior;
3. Absorb base Candidate revision, stale diff behavior, Reject Diff vs Discard Item, and partial-use semantics;
4. minimum Freeze Review Snapshot contents;
5. Review Note default `open` state;
6. cold-start handoff as evidence + Human classification;
7. Human-only Promote and exact Accepted Spec record.

The following remain intentionally deleted/deferred:

- Section-local durable staging ownership;
- normative `◈` / Markdown Capture protocol;
- automatic Work Item inference;
- Review Staging durable object;
- full Review Issue tree / lineage / auto re-review;
- separate Review-ready governance gate;
- Artifact promotion workflow;
- Coding runtime handoff orchestration.

## Primary attack questions

### A. Scope / stale semantics

Can any pending operation created under Work Item A still mutate or be silently retargeted into Work Item B after switching?

Does the stale rule accidentally make harmless operations unnecessarily cumbersome?

### B. Inbox minimum contract

Is newest-first + pending/selected/absorbed + source reference enough to prevent the v0 Inbox from immediately becoming a junk drawer without reintroducing taxonomy burden?

### C. Absorb transaction

Look for counterexamples involving:

- Candidate changes after diff generation;
- selected-but-unused items;
- rejecting a synthesis while retaining source material;
- stale diff acceptance;
- partial use of selected material;
- Candidate/Inbox mutation ordering.

Does v3 now define the product meaning without drifting into low-level merge implementation?

### D. Freeze reproducibility

Is the minimum Review Snapshot sufficient for an independent Reviewer to know exactly what was reviewed and what was intentionally excluded?

Is any listed snapshot field unnecessary ceremony?

### E. Cold-start readiness evidence

Does the procedure distinguish implementation questions from genuine product blockers well enough?

Can a weak implementer still produce a false sense of safety? If yes, is that an acceptable evidence limitation or a blocking flaw?

### F. Human Promote

Is Human-only Promote enough to make `implementation-safe` authoritative without creating a new governance system?

Can the exported Spec drift from the promoted Candidate revision under the stated semantics?

## Simplification challenge

Try to remove another 25–50% of v3. Only recommend removal where the same core user value and safety remain.

Pay special attention to whether:

- Inbox filters can be simplified further;
- Freeze Snapshot contains too much metadata;
- cold-start evidence can be lighter;
- Human Promote record can be smaller.

## Required output

```yaml
review:
  target: "NOOS Deliberation Harness — Review Candidate v3"
  overall_assessment:
  promotion_recommendation:
    first_implementation_slice:
    implementation_safe_now:
  issues:
    - id:
      severity: blocker | major | minor
      target_section:
      finding:
      why_it_matters:
      evidence_or_counterexample:
      minimal_fix_direction:
  simplification_challenge:
    removable_mechanisms:
    irreducible_core:
  strengths_that_should_not_be_lost:
```

Focus on product semantics and workflow boundaries, not implementation technology.
