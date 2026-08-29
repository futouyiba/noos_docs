# NOOS Deliberation Harness — Review Candidate v3

> Status: narrow revision after independent audit of v2. This revision preserves the simplified v0 and only tightens operation ownership, Absorb safety, Freeze reproducibility, readiness evidence, and Human promotion authority.
>
> GitBook frozen revision: `fggdIa5FgR5Xcj6N9Zg9`

## 1. v0 product boundary

The v0 deliberation loop is intentionally small:

```text
Conversation
→ Active Work Item
→ optional AI Capture Proposal / Manual Capture
→ Work Item Inbox
→ Absorb Proposed Diff
→ Human accepts into Working Candidate
→ Freeze Review Snapshot
→ immutable Review Report
→ selected Review Notes
→ one Readiness Checklist
→ Human Promote to implementation-safe
→ GitHub Markdown / Notion Spec
```

The following remain out of scope for v0:

- Section-local durable staging ownership;
- multi-Work-Item binding in one conversation at the same time;
- automatic Work Item inference;
- normative inline marker syntax;
- Artifact promotion workflow;
- Review Issue tree / full lineage / auto re-review;
- separate Review-ready governance gate;
- Coding Agent runtime/session orchestration.

## 2. Explicit Active Work Item and stable operation ownership

### 2.1 Engagement boundary

Harness deliberation is explicit. A conversation may be in ordinary-chat mode or bound to one Active Work Item.

At any moment, one conversation has at most one Active Work Item.

Activating a Work Item means either:

1. create a new Work Item and make it active; or
2. select an existing Work Item and make it active.

Switching does not migrate any historical Capture, Inbox item, Proposed Diff, Review Report, or Review Note.

### 2.2 Creation-time scope binding

Every durable or actionable operation created inside Harness mode is bound to the Work Item that was active when that operation was created.

At minimum this applies to:

- AI Capture Proposal;
- Manual Capture;
- Inbox item;
- Absorb Proposed Diff;
- Freeze Review Snapshot;
- Review Report;
- Review Note.

The current page state at click time must never silently retarget an existing operation.

### 2.3 Stale-after-switch rule

If the user switches from Work Item A to Work Item B while an operation created under A is still pending, that operation remains owned by A.

For interactive pending operations such as Capture Proposal or Absorb Proposed Diff, the UI must treat the operation as stale in the new context and require explicit re-entry/reconfirmation under its original Work Item before applying it.

Example:

```text
Active A
→ AI Capture Proposal P(A)
→ switch to B
→ user clicks Accept on P(A)
```

The system must not write into B. P(A) remains owned by A and must be reopened/reconfirmed in A or regenerated.

This rule is a product semantic invariant, not an implementation-specific concurrency mechanism.

## 3. Capture baseline

Manual Capture remains the reliable baseline.

AI Capture Proposal remains optional and experimental.

Hard requirements:

- Capture never blocks the main conversation;
- absence of AI Capture never blocks discussion;
- marker/parser failure is silently ignorable;
- Capture does not modify Working Candidate;
- Capture belongs to the creation-time Work Item;
- user can continue chatting without immediately Accepting / Rejecting a proposal.

No normative `◈`, Markdown block-binding rule, or per-turn Capture limit is part of the semantic contract.

## 4. Work Item Inbox

Each Work Item has one durable Inbox for captured material.

The Inbox is not section-owned. Section or target-document information is optional routing metadata only.

Minimum v0 retrieval behavior:

- default ordering: newest Capture first;
- filter by `pending`, `selected`, `absorbed`;
- show source reference;
- show optional target hint when present.

Search, semantic indexing, advanced labels, and multi-section ownership are deferred.

Inbox material does not become Current Truth until it is accepted through Absorb.

## 5. Absorb transaction

Absorb is the only v0 path by which Inbox material may propose a change to Working Candidate.

### 5.1 Proposed Diff identity

Each Proposed Diff must bind to:

- Work Item ID;
- base Candidate revision ID;
- selected Inbox item IDs;
- generated proposed Candidate change.

### 5.2 Base-revision safety

A Proposed Diff is only directly applicable while its base Candidate revision is still current.

If Working Candidate changes after the Proposed Diff is generated, the diff becomes stale.

A stale diff must never be silently applied over the newer Candidate.

v0 behavior may be either:

- regenerate against the current Candidate; or
- explicitly ask the user to re-confirm/rebase.

The implementation may choose the UX, but silent overwrite is forbidden.

### 5.3 Distinguish Diff rejection from material discard

The following are separate user meanings:

- **Reject Diff**: reject this synthesis proposal only; source Inbox items remain pending unless the user separately changes them.
- **Discard Item**: explicitly remove/close an Inbox item as no longer useful.
- **Accept Diff**: update Working Candidate to the accepted revision and mark only Inbox items actually used by the accepted diff as absorbed.
- **Unused selected items**: remain pending.

Cancel, failure, stale detection, or rejected diff must not silently mutate Candidate or Inbox statuses.

### 5.4 Atomic semantic boundary

Only Human acceptance of a non-stale Proposed Diff creates the Candidate mutation boundary.

Conceptually:

```text
base Candidate revision
+ selected Inbox items
→ Proposed Diff
→ Human Accept
→ new Working Candidate revision
+ absorbed status for actually used items
```

## 6. Freeze Review Snapshot

Freeze is a version action, not a separate governance gate.

It creates a reproducible immutable Review Snapshot.

The minimum snapshot must fix:

1. Candidate body;
2. Candidate revision ID;
3. current Readiness Checklist result;
4. Open Questions and their current classifications when classified;
5. explicitly included supporting references;
6. counts of relevant excluded/unabsorbed material, at least:
   - pending Inbox items;
   - unresolved/open Review Notes when applicable.

The snapshot must make inclusion and exclusion visible enough that a Reviewer can tell what the Candidate claims and what remained outside the review target.

`Relevant baseline` may vary in representation, but the product semantics above define the minimum included scope.

The Reviewer Packet is execution provenance and must itself be version-bound or content-hashed for a given Review Assignment. It is not part of Candidate identity.

## 7. Review Return

Reviewer submits one immutable Review Report bound to the frozen Review Snapshot.

The Review Report does not modify Working Candidate.

Designer may select only the findings that require durable follow-up and promote them to Review Notes.

A Review Note has the minimal lifecycle:

```text
open
→ addressed | dismissed | deferred
```

A newly selected Review Note always begins as `open`.

Each Note references:

- source Review Report;
- reviewed Candidate revision;
- finding text or source location;
- current disposition/status.

No full issue tree, auto extraction of all Reviewer prose, or re-review lineage is required in v0.

## 8. Lazy Open Question classification

Open Questions are not classified turn-by-turn.

Classification is proposed in batches at boundaries such as Freeze or Readiness assessment.

v0 classes:

1. **Decide now** — unresolved product choice that would force implementation to choose observable behavior, authority, or failure semantics.
2. **Leave to implementation** — technical discretion that does not change product semantics.
3. **Defer with trigger** — product question intentionally postponed; must include a reopening trigger/condition.
4. **Needs human/evidence** — requires authority choice, evidence, experiment, or external fact before resolution.

A deferred question without a reopening trigger is not considered fully classified.

## 9. One Readiness Checklist, two labels

NOOS uses one checklist with two derived semantic labels:

- `reviewable`: coherent enough to freeze and attack;
- `implementation-safe`: sufficiently product-decision-complete for implementation autonomy.

Freeze may happen even with warnings or unresolved questions.

Only `Decide now` / Product Blocking questions hard-block `implementation-safe`.

Minimum checklist dimensions:

- problem and user outcome are explicit;
- happy-path observable behavior is explicit;
- key semantic/authority boundaries are explicit;
- failure behavior that changes product meaning is explicit;
- Open Questions are classified sufficiently for handoff;
- Functional Acceptance Scenarios exist;
- Experience Acceptance Scenarios exist;
- cold-start findings have been reviewed and classified;
- no unresolved `Decide now` product blockers remain.

## 10. Decision Boundary

The final Spec should explicitly separate:

### Decided here

Product-semantic and interface-contract choices that implementation must preserve.

### Leave to implementation

Technical choices where multiple implementations remain product-equivalent.

### Deferred with trigger

Known product questions outside the current slice, each with an explicit reopening condition.

### Needs human/evidence

Items not safe to delegate to implementation and not yet resolved.

A useful test is:

> If two technically reasonable implementations would produce materially different observable user behavior, authority, or failure semantics, the choice is not implementation discretion.

## 11. Cold-start Handoff Test as evidence

The cold-start test is evidence, not an automatic gate and not an Agent authority transfer.

Procedure:

1. provide only the candidate Spec and its explicit supporting references to an implementer/implementation Agent that did not participate in deliberation;
2. ask it to list questions it believes must be answered before implementation and any places where two technically reasonable implementations would create different observable behavior, authority, or failure semantics;
3. collect the findings without treating them as automatically valid;
4. Designer/Human filters out pure implementation choices and maps remaining findings into the four Open Question classes;
5. Human determines whether any unresolved `Decide now` blocker remains.

A failing test must leave a concrete findings list, not only `pass/fail`.

A passing test is supporting evidence for readiness, not authority to Promote.

## 12. Human Promote and implementation-safe record

Only Human can execute Promote to `implementation-safe` in v0.

Agent/Designer may recommend readiness but cannot grant it.

Promotion fixes an Accepted Spec record containing at minimum:

- promoted Candidate revision ID;
- Work Item ID;
- Readiness Checklist result;
- classified Open Questions;
- cold-start findings and their dispositions/classifications;
- Functional Acceptance Scenarios;
- Experience Acceptance Scenarios;
- explicit Decision Boundary;
- Human promotion event/approval.

`implementation-safe` therefore means:

> Human has accepted the recorded readiness evidence and confirmed that no unresolved product blocker remains for this implementation slice.

The exported GitHub Markdown / Notion Spec must correspond to that promoted Candidate revision; Promote is not merely a loose UI badge.

## 13. Acceptance Scenarios for v0

### Functional

1. User starts Work Item A, captures material manually, switches to B, then reopens A; A's Inbox material remains bound to A.
2. AI Capture Proposal created under A cannot be silently accepted into B after switching.
3. Rejecting an Absorb Diff leaves the selected source Inbox items pending.
4. A Proposed Diff generated from Candidate r3 becomes stale after Candidate advances to r4 and cannot silently overwrite r4.
5. Accepting a valid Proposed Diff creates a new Candidate revision and marks only actually used Inbox items absorbed.
6. Freeze produces a reproducible snapshot with explicit included/excluded material semantics.
7. Reviewer output remains an immutable Report and does not directly mutate Candidate.
8. A selected Review Note begins `open` and can move only to addressed/dismissed/deferred in v0.
9. Promote cannot complete while an unresolved `Decide now` blocker remains.
10. Exported implementation-safe Spec references the exact promoted Candidate revision.

### Experience

1. A user can conduct a 30–60 minute design discussion without having to process every Capture proposal immediately.
2. Harness engagement is explicit and ordinary ChatGPT conversation is unaffected outside an Active Work Item.
3. A Work Item Inbox remains usable without requiring section routing or taxonomy maintenance.
4. Capture/Review bookkeeping does not require maintaining duplicate durable objects for the same semantic item.
5. A new implementer can identify the intended product behavior from the promoted Spec without reconstructing the original chat history.

## 14. v0 irreducible core

The v0 core is deliberately limited to:

```text
Explicit Active Work Item
+ Manual Capture / optional non-blocking AI Capture
+ one Work Item Inbox
+ Absorb Proposed Diff with base-revision safety
+ mutable Working Candidate
+ immutable Freeze Review Snapshot
+ immutable Review Report + selected Review Notes
+ lazy Open Question classification
+ one Readiness Checklist
+ cold-start evidence
+ Human Promote to exact implementation-safe Spec revision
```

The product principle remains:

> NOOS should remove deliberation coordination tax, not replace it with Harness bookkeeping.
