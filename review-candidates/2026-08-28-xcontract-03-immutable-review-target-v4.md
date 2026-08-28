# Review Candidate: XCONTRACT-03 — Immutable Review Target Semantics v4

- status: review_candidate
- date: 2026-08-28
- source_role: Design Thread
- supersedes_review_target: `3ef49e10eb1c1e0ef2f014595f589c772fdca453` for XCONTRACT-03 only
- scope: immutable review target, run-scoped State revision identity, snapshot capture, candidate revision identity, target-semantic reference closure, freshness semantics
- explicitly_out_of_scope: Continuation, Rollover, Execution Journal

## Review feedback incorporated

This revision preserves all v3 semantics and makes two narrow identity clarifications:

1. Canonical State revision identity is explicitly Run-scoped:

   ```text
   State Revision Identity = (run_id, state_version)
   ```

   `state_version` remains the Baseline version token, but it is only meaningful within its owning Run. `state_revision_id`, if retained, is an optional alias that MUST resolve to the same `(run_id, state_version)` pair.

2. A content hash used in a frozen target-semantic reference is sufficient only when the exact frozen content remains resolvable/preserved by that hash or by immutable storage associated with it. Hash validation against a mutable locator alone detects drift but does not freeze/recover the original target content.

No Runtime Object type, global State revision system, or mutation authority is added.

---

## 1. Identity model

The review target has two independent identity dimensions:

```text
Review Target
= Canonical State Revision
× Immutable Candidate Revision
```

The Canonical State Revision is Run-scoped:

```text
Canonical State Revision Identity
= (run_id, state_version)
```

This follows the Baseline ownership hierarchy:

```text
Project
  -> Run
      -> Run State
```

Therefore these are different immutable State revisions:

```text
(RUN-A, 37)
(RUN-B, 37)
```

`state_version = 37` alone is not a globally unique State target.

For the Candidate side, three concepts are distinct:

```yaml
candidate_id:
  meaning: logical candidate lineage identity

candidate_revision_id:
  meaning: immutable logical/version identity of one revision

candidate_content_hash:
  meaning: integrity and content-equivalence identity of the immutable revision's target-semantic payload
```

Invariants:

- Two different revision IDs remain two different revisions even if their content hashes are equal.
- Equal content does not collapse revision identity.
- A revert creates a new revision rather than resurrecting the old revision identity.
- A substantive edit never mutates an already reviewable revision in place.
- Candidate Revision immutability includes the transitive closure of all external content that contributes to Review Target semantics.
- State revision equality requires both `run_id` equality and `state_version` equality.

Example:

```text
C5  hash=abc
 ↓ edit
C6  hash=def
 ↓ revert
C7  hash=abc
```

`C5 != C7` as revision identity, although their canonical target-semantic payloads are content-equivalent.

---

## 2. Review Snapshot contract

A `ReviewSnapshot` is an immutable artifact that permanently binds one exact Run-scoped State revision to one exact Candidate revision.

Minimum identity:

```yaml
review_snapshot_id: required

state_target:
  run_id: required                   # owning Baseline Run
  state_version: required            # authoritative version token within that Run
  state_revision_id: optional        # alias only; if present MUST resolve to the same pair

candidate_target:
  candidate_id: required
  candidate_revision_id: required
  candidate_content_hash: required
```

The authoritative State target identity is:

```text
(state_target.run_id, state_target.state_version)
```

`state_revision_id` and `(run_id, state_version)` MUST NOT form independent State identities. If `state_revision_id` is present, it MUST resolve to exactly the same immutable canonical State Revision identified by `(run_id, state_version)`. Any mismatch is invalid Snapshot data.

Examples of invalid or ambiguous State targets:

```yaml
# ambiguous across Runs
state_target:
  state_version: 37

# internally inconsistent alias
state_target:
  run_id: RUN-A
  state_version: 37
  state_revision_id: alias-resolving-to-RUN-B-v37
```

The Snapshot must never contain a live target reference such as:

```text
candidate = latest
candidate = current_head
state = current
run_state = current_without_run_scope
```

Once created, resolving the Snapshot must always return the same target identities.

A `ReviewIssue` references `review_snapshot_id`; it does not silently retarget itself when Candidate head or the owning Run State advances.

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

### 3.1 Target-semantic transitive immutability

Candidate Revision immutability is not limited to its inline bytes.

If Candidate semantics depend on external content, the reviewable revision MUST freeze that dependency to immutable/versioned content before the revision can be used as a Review Target.

Allowed target-semantic reference forms include:

```text
immutable artifact revision ID
content-addressed artifact whose frozen bytes/content are resolvable by address
artifact_id + immutable artifact_revision_id
artifact_id + content hash backed by storage that preserves/resolves that exact immutable content
materialized/frozen copy created from an otherwise mutable source
```

A content hash is not by itself proof that the original content remains reviewable. If a hash participates in a frozen reference, one of the following MUST hold:

```text
1. the hash is a content-addressable locator that can retrieve the exact frozen content;
2. immutable storage preserves the exact content revision associated with that hash;
3. the content was materialized/frozen elsewhere and the hash verifies that preserved immutable copy.
```

This is insufficient:

```text
artifact_id = A17
content_hash = h1

resolution procedure:
  fetch current A17
  verify current bytes hash to h1
```

If `A17` changes from v1 to v2, that procedure can detect that current content no longer matches `h1`, but it cannot recover v1. It is an integrity guard against a mutable locator, not an immutable frozen reference.

Forbidden authoritative target-semantic references include unresolved mutable selectors such as:

```text
artifact = latest
artifact = current
artifact = branch_head
artifact = mutable path
artifact = mutable logical ID with no immutable revision/content resolution
```

A mutable source may be used during drafting, but finalizing a reviewable Candidate Revision MUST resolve/materialize it into an immutable reference and persist that frozen reference in the Candidate revision payload.

The rule applies transitively. If a target-semantic referenced artifact itself depends on other external target-semantic content, those dependencies must likewise resolve to immutable/versioned content, or the system must materialize an equivalent frozen closure.

Therefore a reviewable Candidate Revision identifies a deterministic and recoverable target-semantic closure rather than only an immutable top-level reference string or a drift-detecting checksum.

---

## 4. candidate_content_hash semantics

`candidate_content_hash` is defined as:

```text
hash(canonicalized immutable target-semantic revision payload)
```

The hash domain MUST include all content whose change can alter what a Reviewer is being asked to judge.

Minimum semantic payload includes:

- the Candidate's substantive structured fields;
- prose/body content belonging to the Candidate revision;
- embedded structured blocks that are part of the Candidate;
- immutable/versioned attachment or artifact reference descriptors when those referenced artifacts are part of Review Target semantics;
- enough immutable content identity for those references to preserve deterministic target meaning;
- explicit target-relevant metadata whose value changes the meaning of the Candidate.

For target-semantic external references, merely hashing a mutable locator string is insufficient.

For example, this is invalid as a reviewable target-semantic payload:

```yaml
attachment_ref: A17   # A17 may resolve to different content over time
```

A valid frozen descriptor is conceptually:

```yaml
attachment_ref:
  artifact_id: A17
  artifact_revision_id: AR3
  artifact_content_hash: sha256:artifact-canon-v1:<digest>
```

Another implementation MAY omit an explicit artifact revision ID when the content hash itself is a resolvable content address or immutable storage otherwise preserves the exact content revision associated with it.

The exact field shape is implementation-defined. The contract requirement is that the descriptor resolves to one recoverable immutable content revision and cannot later drift while the Candidate revision identity and hash remain unchanged.

Hash validation against whatever bytes a mutable locator returns at review time does not satisfy this requirement by itself.

The hash domain MUST exclude storage/incidental representation details that do not change Candidate meaning, such as:

- map/object field ordering after canonicalization;
- platform-specific newline representation;
- repository storage location when location is not semantic;
- transport timestamps or retrieval metadata that are not part of Candidate meaning.

### 4.1 Self-describing hash identity

A persisted content hash MUST identify both its digest algorithm and canonicalization/hash-domain profile.

Conceptually:

```text
<algorithm>:<canonicalization-profile>:<digest>
```

Example only:

```text
sha256:candidate-canon-v1:<digest>
```

The contract does not require SHA-256 or a particular serialization. It requires that two hashes are compared as content-equivalence identities only when their schemes/profiles are compatible under the implementation's declared comparison rules.

Hash roles are limited to:

1. integrity verification — persisted/retrieved target-semantic content still corresponds to the immutable revision payload/closure;
2. content-equivalence detection — two different Candidate revisions may contain equivalent canonical target-semantic payloads.

Hash equality MUST NOT be used as revision identity.

A hash used inside a frozen external-content reference additionally needs resolvability/preservation as defined in §3.1; integrity equality alone does not create immutable storage.

---

## 5. Snapshot capture semantics

v0 does NOT claim a cross-store atomic point-in-time transaction between State Store and Artifact Store.

The required guarantee is narrower and stronger where it matters:

> A Snapshot records exactly which immutable Run-scoped State revision `S = (run_id, state_version)` and immutable Candidate revision `C` were captured, and never re-resolves either target or any target-semantic dependency to `latest` during or after creation.

### 5.1 Explicit-target capture

If caller already supplies immutable target identities:

```yaml
run_id: RUN-12
state_version: 37
candidate_revision_id: C8
```

creation validates that `(RUN-12, 37)` identifies an existing immutable canonical State revision, validates that Candidate revision `C8` exists and its target-semantic closure is frozen, reads the required metadata/hash, and persists the exact pair:

```text
S = (RUN-12, 37)
C = C8
```

No head lookup is necessary.

Supplying `state_version` without its owning `run_id` is not a valid explicit State target.

### 5.2 Current-head capture

If caller asks to review "the current Candidate against the current State", the State lookup MUST be scoped to a known Run. A caller or surrounding Run context therefore supplies `run_id` before State head resolution.

Conceptually:

```text
1. establish target run_id = R
2. resolve current state head for R -> S = (R, state_version)
3. resolve current candidate head -> C
4. materialize/verify immutable S and C
5. verify/freeze C's target-semantic external-reference closure
6. optionally verify current heads are still S and C
7. persist ReviewSnapshot(target = S, C)
```

The optional head verification is a conservative precondition check intended to reduce accidental stale-at-birth reviews.

It is NOT an atomic cross-store snapshot guarantee.

A head may still move between verification and Snapshot persistence.
That does not corrupt Snapshot identity because the Snapshot persists the already-resolved immutable pair `(S, C)`, where `S` contains its owning `run_id`, and `C`'s target-semantic closure is itself immutable.

If policy requires "must still be current at creation", a failed optimistic verification aborts and retries with newly resolved heads for the same target Run unless the caller explicitly chooses a different Run.

v0 does not require proving that S and C were simultaneously heads at one physical instant.

### 5.3 No re-resolution rule

After State target resolution has produced `S = (R, V)`, later creation steps MUST NOT reinterpret it as `current_state` or as `state_version = V` without preserving `R`.

After Candidate resolution has produced `C`, later creation steps MUST NOT reinterpret the target as `candidate_head(CandidateId)`.

Likewise, after a target-semantic external reference has been frozen to an immutable artifact/content revision, later creation or review execution MUST NOT reinterpret it through a mutable logical reference such as `latest/current/path`.

This prevents all of the following:

```text
RUN-A/v37 silently re-resolved as RUN-B/v37
metadata from C5 + content from C8
C5 / hash abc + mutable attachment A17 that silently changes v1 -> v2
```

---

## 6. Reviewer Input Provenance is separate from Review Target identity

Reviewer-specific input such as:

- Context Projection identity;
- Review Dimension;
- reviewer prompt/version;
- provider/model metadata;
- Evidence/Attachments supplied only as reviewer execution input rather than Candidate semantics;

may be recorded as provenance of the review execution.

They do not become part of the Review Target identity unless the product explicitly defines them as substantive Candidate target content.

Therefore:

```text
Review Target Identity != Reviewer Execution Context
```

This distinction also determines attachment treatment:

```text
attachment affects Candidate meaning
-> part of target-semantic immutable closure and Candidate hash domain

attachment only informs this Reviewer execution
-> Review Execution Provenance, not Candidate identity/hash
```

This prevents execution-environment changes from manufacturing a fake Candidate revision while still preventing target-semantic attachment drift.

`run_id` is not Reviewer Execution Provenance in this contract. It is part of the canonical State target identity because Run owns Run State in the Baseline object model.

---

## 7. Freshness semantics

Freshness compares the immutable Snapshot target to current heads along two target axes.

### 7.1 State relation

The owning Run is fixed by the Snapshot. State freshness is evaluated against the current canonical State head of that same Run:

```yaml
STATE_EXACT:
  when: current_state(run_id = snapshot.run_id).state_version == snapshot.state_version

STATE_ADVANCED:
  when: current_state(run_id = snapshot.run_id).state_version != snapshot.state_version
```

The comparison is therefore not:

```text
current globally active Run's version
vs
snapshot.state_version
```

and it is not valid to compare `state_version` across different Runs as if they were the same revision lineage.

If an implementation retains `state_revision_id`, it is an alias/check value only and MUST resolve consistently with the same `(run_id, state_version)` pair.

This revision does not add a new freshness disposition for "different active Run". A different Run is simply not the owning State lineage of this Snapshot and is outside this State freshness comparison.

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
  when: compatible_hash_scheme(current_head_content_hash, snapshot.candidate_content_hash)
        and current_head_content_hash == snapshot.candidate_content_hash

CONTENT_DIFFERENT:
  when: compatible schemes and hashes differ

CONTENT_COMPARISON_UNAVAILABLE:
  when: schemes/profiles are not directly comparable
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

The State half of these classifications always refers to the current State head of `snapshot.state_target.run_id`.

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

- Candidate Revision — immutable artifact whose target-semantic external-reference closure is also immutable/versioned;
- Review Snapshot — immutable artifact;
- Review Issue — immutable artifact referencing Snapshot;
- candidate head — mutable orchestration/artifact relation, not a new authority source.

`run_id` and `state_version` are reused from the Baseline Run / Run State model. Their pairing in ReviewSnapshot does not create a new State object or global revision namespace.

Creating any Review/Candidate artifact does not mutate canonical Run State unless a separate State Delta Proposal is authorized and reduced.

---

## 11. Rationale

1. `candidate_revision_id` and `candidate_content_hash` solve different problems. Revision ID answers "which version?"; hash answers "what canonical target-semantic content?". Conflating them recreates the stale-target ambiguity XCONTRACT-03 was meant to remove.
2. Top-level Candidate immutability is insufficient if target-semantic external references can drift. Review reproducibility requires transitive immutable target closure.
3. A checksum against a mutable locator can detect mutation but cannot necessarily recover the original content. Frozen target semantics therefore require immutable/content-addressable resolvability or preserved immutable storage, not merely later hash validation.
4. Baseline State versioning is Run-scoped. `(run_id, state_version)` is the complete State revision identity; preserving only `state_version` would make Snapshot identity ambiguous across Runs.
5. The Review contract needs precise target identity, not distributed-transaction theater. Exact immutable `(S, C)` capture is sufficient for reproducible review; simultaneous-head atomicity is not required in v0.
6. Separating content equivalence from revision freshness preserves provenance while still allowing future policy to exploit no-op/revert equivalence.
7. Defining the hash domain semantically prevents both false negatives (meaningful attachment/field changes omitted) and false positives (newline/field-order representation changes).
8. Self-describing hash identity prevents future algorithm/canonicalization migrations from creating ambiguous equality comparisons.
9. Keeping `state_version` authoritative within its owning Run prevents XCONTRACT-03 from inventing a second independent State revision system while preserving the Baseline's existing Run ownership.

---

## 12. Alternatives considered

### Hash equality defines TARGET_EXACT

Rejected. It collapses distinct revisions after revert or regeneration.

### Candidate payload may reference mutable target-semantic attachments by stable logical ID

Rejected. Stable reference syntax does not imply stable referenced content. A mutable `A17` can change while Candidate revision ID and top-level payload hash remain unchanged.

### Hash the mutable locator and trust later resolution

Rejected. This proves only that the locator string is unchanged, not that Review Target semantics are unchanged.

### Treat artifact_id + expected content hash as frozen even when only current artifact bytes are retrievable

Rejected. That can detect drift but cannot reproduce the reviewed content after mutation. The frozen content must remain resolvable by hash/revision or be preserved by immutable materialization.

### Inline/materialize every attachment byte into the Candidate body

Not required. The contract requires immutable/versioned target-semantic closure, not one physical storage representation. Immutable artifact revisions, content-addressed references, or frozen materialization are sufficient.

### Cross-store transaction required for Snapshot creation

Rejected for v0. It adds infrastructure complexity without improving immutable target reproducibility.

### Put Context Projection into Review Target identity

Rejected. Reviewer execution input/provenance is not Candidate identity.

### Omit content hash entirely

Rejected. Revision ID alone is enough for target identity but does not provide integrity/content-equivalence semantics useful for storage verification and later revalidation optimization.

### Treat state_revision_id and state_version as independent required identities

Rejected. Baseline already provides State versioning; XCONTRACT-03 must not create a second State identity axis. `state_version` remains authoritative within the owning Run, and any additional revision ID is an alias only.

### Treat state_version as globally unique across all Runs

Rejected. Baseline owns Run State under Run and uses `run_id + base_state_version` to locate State basis. XCONTRACT-03 should preserve that ownership instead of inventing a global State-version namespace.

---

## 13. Assumptions

- Canonical State exposes immutable Baseline `state_version` identity scoped to an owning `run_id`.
- `(run_id, state_version)` identifies one immutable canonical Run State revision.
- Candidate revisions can be persisted immutably before review begins.
- Target-semantic external references can be resolved to immutable/versioned content or materialized into an immutable closure before review.
- A hash-only frozen reference is used only where exact content remains resolvable/preserved, not merely to validate current bytes at a mutable locator.
- `candidate_head` can advance independently from canonical Run State.
- Snapshot and Candidate artifacts can reference stable IDs across stores.
- v0 may perform optimistic head verification but has no cross-store transaction requirement.

---

## 14. Remaining open questions

None blocks XCONTRACT-03 promotion from the Design Thread's perspective.

Implementation may still choose:

- concrete `candidate_revision_id` format;
- concrete external artifact revision/reference descriptor;
- canonical serialization format;
- hash algorithm and canonicalization profile naming;
- whether immutable reference closure is represented recursively, by a manifest, or by content-addressed graph/root;
- whether current-head capture performs one or more optimistic verification passes.

Those choices must preserve the invariants above.

---

## 15. Proposed state changes

- Promote the invariant that Review Snapshot binds both an exact Run-scoped immutable State revision and an immutable Candidate revision.
- Define Canonical State Revision identity for Review as `(run_id, state_version)`, reusing Baseline Run ownership and State versioning.
- Keep any `state_revision_id` only as an optional alias that must resolve to the same `(run_id, state_version)` pair.
- Promote transitive target-semantic immutability: every external reference contributing to Candidate Review Target semantics resolves to immutable/versioned content or a materialized frozen equivalent.
- Require any hash-based frozen reference to preserve/resolvably address the exact immutable referenced content; checking a hash against a mutable locator alone is insufficient.
- Promote `candidate_revision_id` as the sole authority for Candidate target freshness/exactness.
- Treat `candidate_content_hash` only as self-describing integrity/content-equivalence identity over canonicalized immutable target-semantic payload/closure.
- Record freshness as State relation within the Snapshot's owning Run × Candidate revision relation; content relation remains diagnostic.
- Explicitly state that v0 Snapshot capture does not guarantee cross-store simultaneous-head atomicity.
- Keep stale/freshness separate from Review Issue disposition.

---

## 16. Review targets for this revision

Independent Reviewer should check only:

1. Does `(run_id, state_version)` completely and minimally identify the Baseline Run State revision without introducing a second State revision system?
2. Are explicit-target capture, current-head capture, no-re-resolution, and State freshness now consistently scoped to the same owning Run?
3. Does the R8 clarification correctly distinguish drift detection from immutable content resolvability/recovery?
4. Did any v3 guarantees regress while making these narrow changes?
5. Is any remaining issue promotion-blocking for XCONTRACT-03?

Requested disposition: review this revision independently; do not infer approval from the Design Thread's incorporation of prior feedback.
