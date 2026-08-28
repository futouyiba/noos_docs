# Review Candidate: XCONTRACT-03 — Immutable Review Target Semantics v2

- status: review_candidate
- date: 2026-08-28
- source_role: Design Thread
- supersedes_review_target: `856837acd93e61d99da9882d2f15ea3cc6c678f2` for XCONTRACT-03 only
- scope: immutable review target, snapshot capture, candidate revision identity, freshness semantics
- explicitly_out_of_scope: Continuation, Rollover, Execution Journal

## Review feedback incorporated

This revision makes one promotion-blocking correction and two narrow contract tightenings:

1. `TARGET_EXACT` is determined by immutable `candidate_revision_id`, never by content-hash equality.
2. Snapshot capture guarantees an exact immutable `(State Revision, Candidate Revision)` pair, not an atomic cross-store observation that both were heads at the same physical instant.
3. `candidate_content_hash` is defined over a canonicalized immutable revision payload and is used for integrity/content equivalence, not revision identity.

No Runtime Object type or mutation authority is added.

---

## 1. Identity model

The review target has two independent identity dimensions:

```text
Review Target
= Canonical State Revision
× Immutable Candidate Revision
```

For the Candidate side, three concepts are distinct:

```yaml
candidate_id:
  meaning: logical candidate lineage identity

candidate_revision_id:
  meaning: immutable logical/version identity of one revision

candidate_content_hash:
  meaning: integrity and content-equivalence identity of the immutable revision payload
```

Invariants:

- Two different revision IDs remain two different revisions even if their content hashes are equal.
- Equal content does not collapse revision identity.
- A revert creates a new revision rather than resurrecting the old revision identity.
- A substantive edit never mutates an already reviewable revision in place.

Example:

```text
C5  hash=abc
 ↓ edit
C6  hash=def
 ↓ revert
C7  hash=abc
```

`C5 != C7` as revision identity, although their canonical payloads are content-equivalent.

---

## 2. Review Snapshot contract

A `ReviewSnapshot` is an immutable artifact that permanently binds one exact State revision to one exact Candidate revision.

Minimum identity:

```yaml
review_snapshot_id: required

state_target:
  state_revision_id: required
  state_version: required

candidate_target:
  candidate_id: required
  candidate_revision_id: required
  candidate_content_hash: required
```

The Snapshot must never contain a live reference such as:

```text
candidate = latest
candidate = current_head
state = current
```

Once created, resolving the Snapshot must always return the same target identities.

A `ReviewIssue` references `review_snapshot_id`; it does not silently retarget itself when Candidate head or Run State advances.

---

## 3. Candidate revision immutability

A revision becomes eligible as a Review Target only after it has been materialized as an immutable revision.

Conceptually:

```text
Candidate logical identity
  C
   ├─ C5 immutable
   ├─ C6 immutable
   └─ C7 immutable

candidate_head(C) -> C7
```

`candidate_head` is a mutable pointer/relation.
`C5`, `C6`, and `C7` are immutable revision identities.

Review binds to `C5`, not to `candidate_head(C)`.

Updating `candidate_head` therefore does not alter an existing Review Snapshot.

---

## 4. candidate_content_hash semantics

`candidate_content_hash` is defined as:

```text
hash(canonicalized immutable revision payload)
```

The hash domain MUST include all content whose change can alter what a Reviewer is being asked to judge.

Minimum semantic payload includes:

- the Candidate's substantive structured fields;
- prose/body content belonging to the Candidate revision;
- embedded structured blocks that are part of the Candidate;
- attachment/artifact references when those referenced artifacts are part of the review target semantics;
- explicit target-relevant metadata whose value changes the meaning of the Candidate.

The hash domain MUST exclude storage/incidental representation details that do not change Candidate meaning, such as:

- map/object field ordering after canonicalization;
- platform-specific newline representation;
- repository storage location when location is not semantic;
- transport timestamps or retrieval metadata that are not part of Candidate meaning.

The concrete serialization and hash algorithm are implementation choices, but the canonicalization contract must make semantically identical immutable payloads hash-equivalent.

Hash roles are limited to:

1. integrity verification — persisted/retrieved content still corresponds to the immutable revision payload;
2. content-equivalence detection — two different revisions may contain equivalent canonical payloads.

Hash equality MUST NOT be used as revision identity.

---

## 5. Snapshot capture semantics

v0 does NOT claim a cross-store atomic point-in-time transaction between State Store and Artifact Store.

The required guarantee is narrower and stronger where it matters:

> A Snapshot records exactly which immutable State revision `S` and immutable Candidate revision `C` were captured, and never re-resolves either target to `latest` during or after creation.

### 5.1 Explicit-target capture

If caller already supplies immutable target identities:

```yaml
state_revision_id: S37
candidate_revision_id: C8
```

creation validates that both immutable revisions exist, reads their required metadata/hash, and persists the exact pair.

No head lookup is necessary.

### 5.2 Current-head capture

If caller asks to review "the current Candidate against the current State", the system may use an optimistic capture protocol:

```text
1. resolve current state head -> S
2. resolve current candidate head -> C
3. materialize/verify immutable S and C
4. optionally verify current heads are still S and C
5. persist ReviewSnapshot(target = S, C)
```

The optional verification is a conservative precondition check intended to reduce accidental stale-at-birth reviews.

It is NOT an atomic cross-store snapshot guarantee.

A head may still move between verification and Snapshot persistence.
That does not corrupt Snapshot identity because the Snapshot persists the already-resolved immutable pair `(S, C)`.

If policy requires "must still be current at creation", a failed optimistic verification aborts and retries with newly resolved heads.
But v0 does not require proving that S and C were simultaneously heads at one physical instant.

### 5.3 No re-resolution rule

After step 2 has resolved `C`, later creation steps MUST NOT reinterpret the target as `candidate_head(CandidateId)`.

Likewise, after resolving `S`, later steps MUST NOT reinterpret the state target as `current_state`.

This prevents a mixed snapshot such as metadata from C5 combined with content from C8.

---

## 6. Reviewer Input Provenance is separate from Review Target identity

Reviewer-specific input such as:

- Context Projection identity;
- Review Dimension;
- reviewer prompt/version;
- provider/model metadata;

may be recorded as provenance of the review execution.

They do not become part of the Review Target identity unless the product explicitly defines them as substantive target content.

Therefore:

```text
Review Target Identity != Reviewer Execution Context
```

This prevents execution-environment changes from manufacturing a fake Candidate revision.

---

## 7. Freshness semantics

Freshness compares the immutable Snapshot target to current heads along two target axes.

### 7.1 State relation

```yaml
STATE_EXACT:
  when: current_state_revision_id == snapshot.state_revision_id

STATE_ADVANCED:
  when: current_state_revision_id != snapshot.state_revision_id
```

`state_version` may serve as the revision token where Baseline defines it as such.

### 7.2 Candidate revision relation

Revision identity is authoritative:

```yaml
TARGET_EXACT:
  when: current_candidate_head_revision_id == snapshot.candidate_revision_id

TARGET_SUPERSEDED:
  when: current_candidate_head_revision_id != snapshot.candidate_revision_id
```

Content hash is NOT part of this equality test.

Therefore the revert example is classified correctly:

```text
snapshot target = C5 / hash abc
current head    = C7 / hash abc

revision_relation = TARGET_SUPERSEDED
content_relation  = CONTENT_EQUIVALENT
```

It is never `TARGET_EXACT`.

### 7.3 Content relation

Content relation is an additional diagnostic fact, orthogonal to revision freshness:

```yaml
CONTENT_EQUIVALENT:
  when: current_head_content_hash == snapshot.candidate_content_hash

CONTENT_DIFFERENT:
  when: current_head_content_hash != snapshot.candidate_content_hash
```

This signal MAY allow later orchestration/adjudication policy to avoid unnecessary re-review in some cases, but this contract does not define that policy.

In particular:

```text
TARGET_SUPERSEDED + CONTENT_EQUIVALENT
```

still means the original Review targeted a superseded revision.

---

## 8. Combined freshness classification

The primary stale/fresh classification uses State relation × Candidate revision relation:

```yaml
EXACT:
  state: STATE_EXACT
  candidate: TARGET_EXACT

STATE_ADVANCED:
  state: STATE_ADVANCED
  candidate: TARGET_EXACT

TARGET_SUPERSEDED:
  state: STATE_EXACT
  candidate: TARGET_SUPERSEDED

BOTH_CHANGED:
  state: STATE_ADVANCED
  candidate: TARGET_SUPERSEDED
```

`content_relation` is attached as diagnostic metadata and does not rewrite these classifications.

Examples:

```yaml
case_revert:
  freshness: TARGET_SUPERSEDED
  content_relation: CONTENT_EQUIVALENT

case_normal_edit:
  freshness: TARGET_SUPERSEDED
  content_relation: CONTENT_DIFFERENT
```

---

## 9. Freshness is not disposition

Freshness is a factual relation between Snapshot target and current heads.

It is not a Review Issue adjudication result.

Forbidden implicit conversions include:

```text
TARGET_SUPERSEDED -> OBSOLETE
STATE_ADVANCED -> REJECT
CONTENT_EQUIVALENT -> ACCEPT
```

Integration/revalidation must inspect whether the reviewed claim still applies to the newer Candidate/State before producing a disposition.

---

## 10. Authority / object boundary

This contract introduces no new Baseline Runtime Object type and no mutation authority.

Durable artifacts/relations:

- Candidate Revision — immutable artifact;
- Review Snapshot — immutable artifact;
- Review Issue — immutable artifact referencing Snapshot;
- candidate head — mutable orchestration/artifact relation, not a new authority source.

Creating any of these artifacts does not mutate canonical Run State unless a separate State Delta Proposal is authorized and reduced.

---

## 11. Rationale

1. `candidate_revision_id` and `candidate_content_hash` solve different problems. Revision ID answers "which version?"; hash answers "what canonical content?". Conflating them recreates the stale-target ambiguity XCONTRACT-03 was meant to remove.
2. The Review contract needs precise target identity, not distributed-transaction theater. Exact immutable `(S, C)` capture is sufficient for reproducible review; simultaneous-head atomicity is not required in v0.
3. Separating content equivalence from revision freshness preserves provenance while still allowing future policy to exploit no-op/revert equivalence.
4. Defining the hash domain semantically prevents both false negatives (meaningful attachment/field changes omitted) and false positives (newline/field-order representation changes).

---

## 12. Alternatives considered

### Hash equality defines TARGET_EXACT

Rejected. It collapses distinct revisions after revert or regeneration.

### Cross-store transaction required for Snapshot creation

Rejected for v0. It adds infrastructure complexity without improving immutable target reproducibility.

### Put Context Projection into Review Target identity

Rejected. Reviewer execution input/provenance is not Candidate identity.

### Omit content hash entirely

Rejected. Revision ID alone is enough for target identity but does not provide integrity/content-equivalence semantics useful for storage verification and later revalidation optimization.

---

## 13. Assumptions

- Canonical State exposes immutable revision identity/version.
- Candidate revisions can be persisted immutably before review begins.
- `candidate_head` can advance independently from canonical Run State.
- Snapshot and Candidate artifacts can reference stable IDs across stores.
- v0 may perform optimistic head verification but has no cross-store transaction requirement.

---

## 14. Remaining open questions

None blocks XCONTRACT-03 promotion.

Implementation may still choose:

- concrete `candidate_revision_id` format;
- canonical serialization format;
- hash algorithm;
- whether current-head capture performs one or more optimistic verification passes.

Those choices must preserve the invariants above.

---

## 15. Proposed state changes

- Promote the invariant that Review Snapshot binds both immutable State revision and immutable Candidate revision.
- Promote `candidate_revision_id` as the sole authority for Candidate target freshness/exactness.
- Treat `candidate_content_hash` only as integrity/content-equivalence identity over canonicalized immutable revision payload.
- Record freshness as State relation × Candidate revision relation; content relation remains diagnostic.
- Explicitly state that v0 Snapshot capture does not guarantee cross-store simultaneous-head atomicity.
- Keep stale/freshness separate from Review Issue disposition.

---

## 16. Review targets for this revision

Independent Reviewer should now check only:

1. Does revision-ID-first freshness fully close the C5 → C6 → C7 revert counterexample?
2. Is the weakened Snapshot capture claim sufficient for reproducibility without accidentally permitting mixed/latest re-resolution?
3. Is the canonical payload hash boundary precise enough for contract level while leaving serialization implementation open?
4. Does any part of this revision accidentally promote Reviewer Input Provenance into Review Target identity?
5. Is any remaining issue promotion-blocking for XCONTRACT-03?

Requested disposition: review this revision independently; do not infer approval from the Design Thread's incorporation of prior feedback.