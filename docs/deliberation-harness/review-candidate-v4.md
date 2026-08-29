# NOOS Deliberation Harness — Review Candidate v4

> Status: narrow revision after v3 independent review. This revision closes the remaining implementation-safety semantics without reopening removed v0 mechanisms.

GitBook frozen revision: `gXJhYz6y0hxslcRqGgcE`

## 1. Scope

v4 keeps the v3 minimal loop:

```text
Explicit Active Work Item
→ Manual / optional AI Capture
→ Work Item Inbox
→ Absorb Proposed Diff
→ Human Accept
→ Working Candidate mutation
→ Freeze immutable Review Snapshot
→ Immutable Review Report
→ selected Review Notes
→ Readiness evidence
→ Human Promote implementation-safe
```

The following remain out of scope for v0: Section-local staging ownership, durable Review Staging, normative marker syntax, full issue lineage, multi-agent orchestration, multiple readiness gates, and Codex runtime management.

## 2. Active Work Item and operation ownership

A Conversation may have at most one **Active Work Item** at a time. Activating a Work Item means selecting an existing Work Item or explicitly creating a new one.

All pending operations are bound at creation time to a stable `work_item_id`:

- Manual Capture;
- AI Capture Proposal;
- Absorb Proposed Diff;
- Freeze / Review assignment.

Switching Active Work Item never migrates an existing operation and never changes its ownership.

### 2.1 Context mismatch, not durable stale state

`stale-after-switch` is **not** a persistent lifecycle status. It is a derived context mismatch:

```text
operation.work_item_id != active_work_item_id
→ operation cannot be applied in current context
```

If the user switches back to the owning Work Item and the operation's other preconditions still hold, it may be confirmed again. No stale history or reopened lifecycle is required.

## 3. Work Item Inbox

The v0 Inbox is one durable staging surface per Work Item.

Minimum durable fields:

- `item_id`;
- `work_item_id`;
- content;
- source reference;
- optional target hint;
- durable state.

Durable Inbox state is intentionally small:

```text
pending
absorbed
closed(reason)
```

`selected` is **not** a durable state. It is only a temporary set within an active Absorb operation.

`deferred`, `rejected`, and `discarded` do not require separate durable workflow objects; when useful they are represented by `closed(reason)`.

Minimum retrieval behavior:

- newest-first by default;
- filter pending / absorbed;
- show source reference;
- show optional target hint.

Search, complex tags and Section ownership remain deferred.

## 4. Capture contract

Manual Capture is the reliable baseline and must work without AI Capture Proposal.

AI Capture Proposal is optional, non-blocking, and may be disabled. Missing or malformed proposal markers must not block Conversation or Candidate work.

Capture never directly mutates the Working Candidate.

## 5. Absorb Transaction

Absorb is the only v0 path from staged material into a Working Candidate.

A Proposed Diff MUST bind:

- `work_item_id`;
- `base_candidate_revision_id`;
- `selected_inbox_item_ids`;
- `incorporated_inbox_item_ids`;
- proposed Candidate diff/content.

The two Inbox sets have distinct semantics:

```text
selected = material supplied to reasoning for this Absorb attempt
incorporated = material the proposed revision claims to have actually absorbed
```

`incorporated_inbox_item_ids` MUST be a subset of `selected_inbox_item_ids`.

If the reasoning worker cannot confidently claim incorporation, the conservative rule applies: **do not list the item as incorporated; leave it pending**.

### 5.1 Accept / Reject / Discard

**Accept Diff** is allowed only when:

- current Active Work Item matches `work_item_id`;
- current Candidate revision equals `base_candidate_revision_id`;
- Human accepts the Proposed Diff.

On successful Accept, atomically:

1. update Working Candidate to the accepted revision;
2. mark only `incorporated_inbox_item_ids` as `absorbed`;
3. leave all selected-but-not-incorporated items `pending`.

If Candidate revision changed since proposal creation, the Diff is not applicable. It must be regenerated or explicitly re-based; v0 does not silently apply an old Diff.

**Reject Diff** rejects only that proposed synthesis. Inbox items remain unchanged.

**Discard Item** is a separate explicit action that closes that Inbox item with a reason.

Cancel/failure before Human Accept produces no Candidate mutation and no absorbed state changes.

## 6. Freeze Review Snapshot

Freeze is a versioning action, not a separate governance gate.

The immutable Review Snapshot MUST identify:

- `snapshot_id`;
- `work_item_id`;
- exact Candidate revision ID;
- Candidate body/content identity;
- readiness checklist at freeze time;
- current Open Question classifications that are intentionally included;
- explicitly included supporting references;
- fixed identity/version for each supporting reference (for example revision, commit, immutable URL, or content hash);
- counts of excluded pending Inbox material and other excluded review material.

The Snapshot need not copy every supporting artifact. It must make the reviewed scope reproducible and make inclusion/exclusion explicit.

Reviewer instructions / packet belong to Review Execution Provenance and must themselves be version-identifiable; they are not part of Candidate identity.

## 7. Review Return

Review produces an immutable Review Report bound to the Review Snapshot.

Designer may elevate only selected findings into lightweight Review Notes.

Review Note state:

```text
open
→ addressed-by-designer
→ dismissed(reason)
→ deferred(reason/trigger)
```

`addressed-by-designer` explicitly does **not** mean Reviewer-verified resolution. v0 does not require a re-review state machine.

Review Report never directly mutates Candidate content.

## 8. Open Questions and blocking semantics

Open Questions are classified lazily at Freeze / implementation-readiness assessment, not turn-by-turn.

v0 uses four classes:

1. `Decide now`;
2. `Leave to implementation`;
3. `Defer with trigger`;
4. `Needs human/evidence`.

Every question evaluated for the current implementation slice also has one semantic readiness property:

```text
blocking: yes | no
```

Rules:

- unresolved `Decide now` that affects the current slice → `blocking: yes`;
- unresolved `Needs human/evidence` that can change observable behavior, authority, or failure semantics of the current slice → `blocking: yes`;
- `Needs human/evidence` may become non-blocking only by being explicitly moved out of the current slice into `Defer with trigger`;
- every deferred item MUST have a reopen trigger;
- `Leave to implementation` is non-blocking only when alternative implementations do not change product semantics visible to users or authority/failure behavior.

Therefore `implementation-safe` cannot be reached while any unresolved current-slice question has `blocking: yes`.

## 9. One Readiness Checklist, two labels

There is one readiness assessment surface with two semantic labels:

- `reviewable`: coherent enough to freeze and attack;
- `implementation-safe`: safe to delegate remaining freedom to an implementation agent.

Freeze for Review is always allowed with warnings; it is not a governance gate.

`implementation-safe` requires at minimum:

- no unresolved current-slice `blocking: yes` questions;
- behavioral and semantic contracts sufficient to prevent implementer product invention;
- explicit Decision Boundary: decided here / leave to implementation / deferred with trigger;
- Functional Acceptance Scenarios;
- Experience Acceptance Scenarios;
- cold-start handoff evidence reviewed by a Human.

## 10. Cold-start Handoff Test

Cold-start is **supporting evidence**, never readiness authority.

Procedure:

1. provide only the candidate implementation Spec to an implementer / Coding Agent that did not participate in deliberation;
2. ask it to identify remaining ambiguities and decisions it would need from product;
3. retain only findings that could alter observable behavior, authority boundaries, or failure semantics;
4. ignore questions that are purely implementation choices and do not alter those semantics;
5. map retained findings into the four Open Question classes;
6. Human confirms whether any retained finding is a current-slice blocker.

A cold-start pass cannot itself grant `implementation-safe`. A failure produces a concrete finding list, not merely pass/fail.

## 11. Human Promote

Only a Human may Promote a Candidate to `implementation-safe`.

Promote is a lightweight authority event, not a new copy of the Spec.

Minimum Promote record:

```text
work_item_id
candidate_revision_id
readiness_checklist_result_or_hash
cold_start_report_reference
human_approval_event
exported_spec_reference (when available)
```

Acceptance Scenarios, Decision Boundary, Open Question details and supporting materials remain read through the immutable Candidate / Snapshot / referenced evidence. Promote does not duplicate those bodies.

Promotion is prohibited while any current-slice Open Question has `blocking: yes`.

## 12. Acceptance Scenarios

### Functional

1. **Cross-Work-Item proposal** — Proposal created in A cannot be applied while B is active; switching back to A restores applicability if its base revision still matches.
2. **Old Absorb Diff** — Candidate changes after Proposed Diff creation; old Diff cannot silently apply.
3. **Partial incorporation** — four items selected, only IDs 1 and 3 are declared incorporated; on Accept only 1 and 3 become absorbed, 2 and 4 remain pending.
4. **Reject Diff** — rejecting synthesis leaves all source Inbox states unchanged.
5. **Discard Item** — explicit discard closes only the chosen item and does not imply rejection of prior/next Diff attempts.
6. **Freeze reproducibility** — Reviewer can identify exact Work Item, Candidate revision, included references and excluded pending-material counts.
7. **Needs human/evidence blocker** — unresolved current-slice behavior question cannot be Promoted until resolved or explicitly deferred with reopen trigger.
8. **Human Promote** — Agent recommendation alone cannot set implementation-safe.

### Experience

1. Normal Conversation remains usable without any AI Capture marker.
2. Work Item switching does not force users to maintain a stale-state workflow.
3. Absorb selection is temporary; canceling an Absorb attempt does not require Inbox cleanup.
4. Review does not require converting every finding into an issue object.
5. Promotion does not require copying the Spec into a new administrative document.

## 13. v0 Irreducible Core

The v0 contract is intentionally limited to:

1. Explicit Active Work Item and creation-time ownership;
2. Manual Capture baseline plus optional non-blocking AI suggestions;
3. one Work Item Inbox;
4. Absorb with base revision safety and explicit incorporated Inbox IDs;
5. immutable reproducible Review Snapshot;
6. immutable Review Report + selected lightweight Review Notes;
7. lazy Open Question classification with explicit blocking semantics;
8. one readiness checklist with reviewable / implementation-safe labels;
9. cold-start evidence + Human classification;
10. Human-only Promote.

No additional durable workflow object should be introduced into the first implementation slice without new evidence.
