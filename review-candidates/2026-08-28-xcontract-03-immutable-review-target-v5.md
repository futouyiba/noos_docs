# Review Candidate: XCONTRACT-03 — Immutable Review Target Semantics v5

- status: review_candidate
- date: 2026-08-28
- source_role: Design Thread
- supersedes_review_target: `798e03e936e87166f2183de6f2353b1612801359` for XCONTRACT-03 only
- scope: immutable review target, run-scoped State revision identity, Candidate finalization boundary, snapshot capture, target-semantic reference closure, freshness semantics
- explicitly_out_of_scope: Continuation, Rollover, Execution Journal

## Review feedback incorporated

This revision preserves all v4 semantics and makes two narrow corrections:

1. **Candidate finalization and Snapshot capture are now strictly separated.**
   Target-semantic dependency freezing occurs before an immutable Candidate Revision exists as a reviewable revision. Snapshot capture only verifies that the selected immutable Candidate Revision is already finalized, frozen, recoverable, and internally consistent. Snapshot capture never mutates or completes Candidate target semantics.
2. **Snapshot field paths are normalized to the schema.**
   State freshness uses `snapshot.state_target.run_id` / `snapshot.state_target.state_version`; Candidate freshness/content diagnostics use `snapshot.candidate_target.*`. No pseudocode shorthand introduces a second field location.

No Runtime Object type, global State revision system, mutation authority, or new review disposition is added.

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
  meaning: immutable logical/version identity of one finalized Candidate revision

candidate_content_hash:
  meaning: integrity and content-equivalence identity of the finalized revision's immutable target-semantic payload
```

Invariants:

- Two different Candidate revision IDs remain two different revisions even if their content hashes are equal.
- Equal content does not collapse revision identity.
- A revert creates a new revision rather than resurrecting the old revision identity.
- A substantive edit never mutates an already finalized Candidate Revision in place.
- A Candidate Revision becomes reviewable only after its complete target-semantic transitive closure has been frozen and made recoverable.
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

A `ReviewSnapshot` is an immutable artifact that permanently binds one exact Run-scoped State revision to one exact finalized Candidate revision.

Minimum identity:

```yaml
review_snapshot_id: required

state_target:
  run_id: required
  state_version: required
  state_revision_id: optional

candidate_target:
  candidate_id: required
  candidate_revision_id: required
  candidate_content_hash: required
```

State target authority:

```text
State Revision Identity
= (state_target.run_id, state_target.state_version)
```

`state_revision_id`, if present, is only an alias/check value. It MUST resolve to exactly the same immutable canonical State Revision identified by `(run_id, state_version)`. Any mismatch is invalid Snapshot data.

Candidate target authority:

```text
Candidate Revision Identity
= candidate_target.candidate_revision_id
```

`candidate_id` identifies the logical Candidate lineage; `candidate_content_hash` provides integrity/content-equivalence identity. Neither replaces `candidate_revision_id` as revision identity.

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
artifact = latest
```

Once created, resolving the Snapshot must always return the same target identities.

A `ReviewIssue` references `review_snapshot_id`; it does not silently retarget itself when Candidate head or the owning Run State advances.

---

## 3. Candidate lifecycle and revision immutability

The Candidate lifecycle has a hard boundary between mutable working material and immutable Candidate Revisions.

Conceptually:

```text
Mutable Candidate Draft
  ↓ finalization
Frozen target-semantic closure
  ↓ canonicalize + hash + persist
Immutable Candidate Revision C
  ↓ eligible
Review Target
```

A mutable draft is not a Candidate Revision merely because it has a logical Candidate ID or temporary storage record.

### 3.1 Candidate finalization

Finalization is the only stage allowed to resolve/materialize mutable target-semantic dependencies into frozen Candidate semantics.

Required conceptual sequence:

```text
1. take mutable Candidate draft D
2. identify all content belonging to Candidate Review Target semantics
3. resolve/materialize/freeze every target-semantic external dependency transitively
4. ensure the resulting closure is immutable/versioned and recoverable
5. persist the frozen reference descriptors into the Candidate payload
6. canonicalize the complete target-semantic payload
7. compute the self-describing candidate_content_hash
8. persist immutable Candidate Revision C with a new candidate_revision_id
9. only now may C become reviewable / candidate revision head
```

If any required target-semantic dependency cannot be frozen/recovered, finalization fails. No reviewable Candidate Revision is produced from that draft state.

Finalization MUST NOT mutate an existing Candidate Revision. If a previously persisted working artifact must change target semantics, the result is a new Candidate Revision identity.

### 3.2 Candidate revision head

Within this contract:

```text
candidate_head(candidate_id)
```

means the head of the **finalized immutable Candidate Revision lineage**.

It does not mean an editor buffer, mutable working draft, or unresolved artifact head.

An implementation MAY maintain a separate mutable `working_draft` / `working_head`, but Snapshot capture cannot bind that mutable object as a Candidate Revision target.

Conceptually:

```text
Candidate logical identity C
  mutable draft D9          # not reviewable, not candidate_head in this contract

  immutable revisions:
    C5
    C6
    C7 <- candidate_head(C)
```

Updating `candidate_head` changes which immutable Candidate Revision is latest; it never changes the content of an existing Candidate Revision.

### 3.3 Target-semantic transitive immutability

Candidate Revision immutability is not limited to inline bytes.

If Candidate semantics depend on external content, the Candidate MUST freeze that dependency during finalization, before the Candidate Revision becomes reviewable.

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

If `A17` changes from v1 to v2, that procedure can detect drift but cannot recover v1. It is an integrity guard against a mutable locator, not an immutable frozen reference.

Forbidden authoritative target-semantic references in a finalized Candidate Revision include unresolved mutable selectors such as:

```text
artifact = latest
artifact = current
artifact = branch_head
artifact = mutable path
artifact = mutable logical ID with no immutable revision/content resolution
```

The rule applies transitively. If a target-semantic referenced artifact itself depends on other target-semantic external content, those dependencies must likewise resolve to immutable/versioned content, or finalization must materialize an equivalent frozen closure.

Therefore a reviewable Candidate Revision identifies a deterministic and recoverable target-semantic closure rather than only an immutable top-level reference string or a drift-detecting checksum.

---

## 4. candidate_content_hash semantics

`candidate_content_hash` is defined as:

```text
hash(canonicalized immutable target-semantic revision payload)
```

The hash is computed only after Candidate finalization has produced the complete frozen target-semantic payload/closure.

The hash domain MUST include all content whose change can alter what a Reviewer is being asked to judge.

Minimum semantic payload includes:

- the Candidate's substantive structured fields;
- prose/body content belonging to the Candidate revision;
- embedded structured blocks that are part of the Candidate;
- immutable/versioned attachment or artifact reference descriptors when those referenced artifacts are part of Review Target semantics;
- enough immutable content identity for those references to preserve deterministic target meaning;
- explicit target-relevant metadata whose value changes the meaning of the Candidate.

For target-semantic external references, merely hashing a mutable locator string is insufficient.

Invalid:

```yaml
attachment_ref: A17
```

Conceptually valid frozen descriptor:

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

Hash equality MUST NOT be used as Candidate revision identity.

A hash used inside a frozen external-content reference additionally needs resolvability/preservation as defined in §3.3; integrity equality alone does not create immutable storage.

---

## 5. Snapshot capture semantics

v0 does NOT claim a cross-store atomic point-in-time transaction between State Store and Artifact Store.

The required guarantee is:

> A Snapshot records exactly which immutable Run-scoped State revision `S = (run_id, state_version)` and already-finalized immutable Candidate revision `C` were captured, and never re-resolves either target or any target-semantic dependency to `latest` during or after creation.

Snapshot capture is a **selection + validation + persistence** operation.

It is NOT a Candidate finalization operation.

Snapshot capture MUST NOT:

```text
resolve a mutable target-semantic dependency into new Candidate semantics
materialize a missing Candidate dependency into the selected revision
rewrite a Candidate external reference
recompute Candidate semantics and keep the same candidate_revision_id
create a Snapshot-only semantic override for an otherwise incomplete Candidate Revision
```

### 5.1 Explicit-target capture

If caller already supplies immutable target identities:

```yaml
state_target:
  run_id: RUN-12
  state_version: 37

candidate_target:
  candidate_revision_id: C8
```

creation performs conceptually:

```text
1. validate S = (RUN-12, 37) exists as immutable canonical State revision
2. load Candidate Revision C8
3. verify C8 is a finalized immutable Candidate Revision
4. verify C8's target-semantic closure is already frozen and recoverable
5. verify its persisted canonical payload/hash metadata is internally consistent
6. persist ReviewSnapshot(target = S, C8)
```

No head lookup and no Candidate semantic mutation are necessary.

If C8 contains an unresolved mutable target-semantic dependency, capture fails with a reviewability failure such as:

```text
CANDIDATE_NOT_REVIEWABLE
```

The caller/orchestrator may separately finalize working material into a new Candidate Revision and then start Snapshot capture again against that new revision.

Supplying `state_version` without its owning `run_id` is not a valid explicit State target.

### 5.2 Current-head capture

If caller asks to review "the current Candidate against the current State", the State lookup MUST be scoped to a known Run. A caller or surrounding Run context supplies `run_id` before State head resolution.

Conceptually:

```text
1. establish target run_id = R
2. resolve current State head for R -> S = (R, state_version)
3. resolve finalized immutable Candidate revision head -> C
4. verify S exists as immutable canonical State revision
5. verify C is already finalized, immutable, and reviewable
6. verify C's target-semantic closure is already frozen and recoverable
7. verify C's persisted payload/hash metadata is internally consistent
8. optionally verify current heads are still S and C
9. persist ReviewSnapshot(target = S, C)
```

There is deliberately no `freeze C` step in Snapshot capture.

If step 5, 6, or 7 fails:

```text
abort Snapshot capture
-> CANDIDATE_NOT_REVIEWABLE (or equivalent validation failure)
```

If mutable working material exists and should become the target, the orchestration layer must first run Candidate finalization, which produces a **new immutable Candidate Revision**, then restart Snapshot capture and resolve/select that revision explicitly or as the new finalized head.

This avoids all three invalid outcomes:

```text
1. mutate selected C in place
2. silently change target C -> C'
3. store semantic overrides only in ReviewSnapshot
```

The optional head verification is a conservative precondition check intended to reduce accidental stale-at-birth reviews.

It is NOT an atomic cross-store snapshot guarantee.

A State or Candidate head may still move between verification and Snapshot persistence. That does not corrupt Snapshot identity because the Snapshot persists the already-resolved immutable pair `(S, C)`.

If policy requires "must still be current at creation", a failed optimistic verification aborts and retries with newly resolved heads for the same target Run unless the caller explicitly chooses a different Run.

v0 does not require proving that S and C were simultaneously heads at one physical instant.

### 5.3 No re-resolution rule

After State target resolution has produced `S = (R, V)`, later creation steps MUST NOT reinterpret it as `current_state` or as `state_version = V` without preserving `R`.

After Candidate resolution has produced `C`, later creation steps MUST NOT reinterpret the target as `candidate_head(candidate_id)`.

After a target-semantic external reference is already frozen inside finalized Candidate Revision `C`, later creation or review execution MUST NOT reinterpret it through a mutable logical reference such as `latest/current/path`.

Snapshot validation may reject `C` if the frozen target cannot be recovered or integrity verification fails. It does not repair `C` in place.

This prevents all of the following:

```text
RUN-A/v37 silently re-resolved as RUN-B/v37
metadata from C5 + content from C8
C5 silently rewritten because A17/latest resolves differently
Snapshot-only override that makes reviewed semantics differ from Candidate C5
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

Attachment treatment remains:

```text
attachment affects Candidate meaning
-> freeze during Candidate finalization
-> part of Candidate target-semantic closure/hash domain

attachment only informs this Reviewer execution
-> Review Execution Provenance
-> not Candidate identity/hash
```

Snapshot capture cannot move an execution-only attachment into Candidate semantics, nor can it use a Snapshot-only attachment to repair incomplete Candidate target semantics.

`run_id` is not Reviewer Execution Provenance. It is part of canonical State target identity because Run owns Run State in the Baseline object model.

---

## 7. Freshness semantics

Freshness compares the immutable Snapshot target to current heads along two target axes.

All field paths below use the schema defined in §2.

### 7.1 State relation

The owning Run is fixed by the Snapshot. State freshness is evaluated against the current canonical State head of that same Run:

```yaml
STATE_EXACT:
  when: current_state(
          run_id = snapshot.state_target.run_id
        ).state_version
        == snapshot.state_target.state_version

STATE_ADVANCED:
  when: current_state(
          run_id = snapshot.state_target.run_id
        ).state_version
        != snapshot.state_target.state_version
```

The comparison is not:

```text
current globally active Run's version
vs
snapshot.state_target.state_version
```

and it is invalid to compare `state_version` across different Runs as if they were the same revision lineage.

If an implementation retains `state_revision_id`, it is an alias/check value only and MUST resolve consistently with:

```text
(
  snapshot.state_target.run_id,
  snapshot.state_target.state_version
)
```

This contract does not add a freshness disposition for "different active Run". A different Run is not the owning State lineage of this Snapshot and is outside this State freshness comparison.

### 7.2 Candidate revision relation

Revision identity is authoritative:

```yaml
TARGET_EXACT:
  when: current_candidate_head_revision_id(
          candidate_id = snapshot.candidate_target.candidate_id
        )
        == snapshot.candidate_target.candidate_revision_id

TARGET_SUPERSEDED:
  when: current_candidate_head_revision_id(
          candidate_id = snapshot.candidate_target.candidate_id
        )
        != snapshot.candidate_target.candidate_revision_id
```

Content hash is NOT part of this equality test.

Therefore:

```text
snapshot.candidate_target = C5 / hash abc
current finalized head    = C7 / hash abc

revision_relation = TARGET_SUPERSEDED
content_relation  = CONTENT_EQUIVALENT
```

It is never `TARGET_EXACT`.

A mutable working draft does not alter this Candidate revision freshness relation until it is finalized into a new immutable Candidate Revision and becomes the finalized Candidate revision head.

### 7.3 Content relation

Content relation is an additional diagnostic fact, orthogonal to revision freshness:

```yaml
CONTENT_EQUIVALENT:
  when: compatible_hash_scheme(
          current_head_content_hash,
          snapshot.candidate_target.candidate_content_hash
        )
        and current_head_content_hash
            == snapshot.candidate_target.candidate_content_hash

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

still means the original Review targeted a superseded Candidate revision.

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

`content_relation` is diagnostic metadata and does not rewrite these classifications.

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

Freshness is a factual relation between Snapshot target and current finalized heads.

It is not a Review Issue adjudication result.

Forbidden implicit conversions include:

```text
TARGET_SUPERSEDED -> OBSOLETE
STATE_ADVANCED -> REJECT
CONTENT_EQUIVALENT -> ACCEPT
CANDIDATE_NOT_REVIEWABLE -> create Snapshot anyway
```

`CANDIDATE_NOT_REVIEWABLE` is a capture/finalization boundary failure, not an Issue disposition.

Integration/revalidation must inspect whether a reviewed claim still applies to newer Candidate/State revisions before producing a disposition.

---

## 10. Authority / object boundary

This contract introduces no new Baseline Runtime Object type and no canonical mutation authority.

Durable artifacts/relations:

- Candidate Revision — finalized immutable artifact whose target-semantic external-reference closure is already immutable/versioned and recoverable;
- Review Snapshot — immutable artifact selecting an existing State revision and existing Candidate Revision;
- Review Issue — immutable artifact referencing Snapshot;
- candidate revision head — mutable orchestration/artifact relation pointing only to finalized Candidate Revisions under this contract.

Mutable working drafts may exist as authoring/orchestration artifacts, but they are not Review Target Candidate Revisions.

Candidate finalization creates a new durable Candidate Revision; it does not mutate canonical Run State.

Snapshot creation likewise does not mutate canonical Run State.

Any canonical Run State mutation still requires a separate State Delta Proposal authorized and reduced through the Baseline authority path.

---

## 11. Rationale

1. `candidate_revision_id` and `candidate_content_hash` solve different problems. Revision ID answers "which finalized version?"; hash answers "what canonical target-semantic content?".
2. Review reproducibility requires transitive immutable and recoverable Candidate target closure.
3. That closure must be established **before** Candidate Revision identity becomes reviewable. Otherwise Snapshot creation would either mutate an immutable revision, silently change the selected revision, or create Snapshot-only target semantics.
4. Snapshot capture therefore has a validation boundary, not a semantic-completion boundary.
5. Exact immutable `(S, C)` capture is sufficient for reproducible review; simultaneous-head atomicity is not required in v0.
6. Separating content equivalence from revision freshness preserves provenance while allowing future policy to exploit no-op/revert equivalence.
7. Self-describing hash identity prevents algorithm/canonicalization migrations from creating ambiguous equality comparisons.
8. `(run_id, state_version)` reuses Baseline Run ownership without inventing a global State revision namespace.
9. Schema-qualified field paths eliminate accidental ambiguity about where authoritative target identity lives.

---

## 12. Alternatives considered

### Freeze Candidate dependencies during Snapshot capture

Rejected. If the selected Candidate Revision is already immutable, changing its target-semantic payload violates immutability. If freezing produces a new Candidate Revision, Snapshot must select that new revision through a new capture attempt rather than silently changing target identity.

### Store frozen dependency overrides only inside ReviewSnapshot

Rejected. This would make Review Target semantics differ from the selected Candidate Revision and violate the invariant that target-semantic references are persisted in the finalized Candidate revision payload.

### Let candidate_head point directly to mutable draft material

Rejected for this contract's `candidate_head` relation. Implementations may maintain a separate working-draft head, but Review Snapshot target resolution operates over finalized Candidate Revision lineage.

### Hash equality defines TARGET_EXACT

Rejected. It collapses distinct revisions after revert or regeneration.

### Candidate payload may reference mutable target-semantic attachments by stable logical ID

Rejected. Stable reference syntax does not imply stable referenced content.

### Hash mutable locator and trust later resolution

Rejected. It detects locator-content mismatch but does not preserve/recover the reviewed content.

### Inline/materialize every attachment byte into Candidate body

Not required. Immutable artifact revisions, content-addressed references, or materialized immutable copies are all acceptable.

### Cross-store transaction required for Snapshot creation

Rejected for v0. It adds infrastructure complexity without improving immutable target reproducibility.

### Put Context Projection into Review Target identity

Rejected. Reviewer execution input/provenance is not Candidate identity.

### Treat state_revision_id and state_version as independent required identities

Rejected. Baseline already provides Run-scoped State versioning; XCONTRACT-03 does not create a second State identity axis.

---

## 13. Assumptions

- Canonical State exposes immutable Run-scoped identity `(run_id, state_version)`.
- Candidate authoring can maintain mutable working material separately from finalized Candidate Revisions.
- Candidate finalization can resolve/materialize target-semantic external references to immutable/versioned and recoverable content before revision persistence.
- `candidate_head` in this contract points to finalized immutable Candidate Revisions.
- Candidate finalized head can advance independently from canonical Run State.
- Snapshot and Candidate artifacts can reference stable IDs across stores.
- v0 may perform optimistic head verification but has no cross-store transaction requirement.

---

## 14. Remaining open questions

None blocks XCONTRACT-03 promotion from the Design Thread's perspective.

Implementation may still choose:

- concrete `candidate_revision_id` format;
- names/storage model for mutable Candidate drafts;
- concrete external artifact revision/reference descriptor;
- canonical serialization format;
- hash algorithm and canonicalization profile naming;
- whether immutable reference closure is represented recursively, by a manifest, or by content-addressed graph/root;
- exact error code spelling for non-reviewable Candidate capture;
- whether current-head capture performs one or more optimistic verification passes.

Those choices must preserve the invariants above.

---

## 15. Proposed state changes

- Promote Review Snapshot identity as Run-scoped immutable State revision `(run_id, state_version)` × immutable finalized Candidate Revision.
- Promote the Candidate lifecycle boundary: target-semantic dependency freeze/materialization happens during Candidate finalization, before immutable Candidate Revision persistence/review eligibility.
- Require Snapshot capture to validate, never mutate or complete, Candidate target semantics.
- Require capture failure when a selected Candidate Revision is not already finalized/frozen/recoverable; any required semantic change must produce a new Candidate Revision and a new capture attempt.
- Preserve transitive target-semantic immutability and recoverability.
- Preserve `candidate_revision_id` as sole authority for Candidate target freshness/exactness.
- Preserve self-describing `candidate_content_hash` only for integrity/content-equivalence identity.
- Preserve Baseline `(run_id, state_version)` State identity; `state_revision_id` remains only a consistent alias.
- Normalize Snapshot field paths to `state_target.*` and `candidate_target.*`.
- Preserve State freshness × Candidate revision freshness; content relation remains diagnostic.
- Preserve no cross-store simultaneous-head atomicity claim.
- Preserve stale/freshness separate from Review Issue disposition.

---

## 16. Review targets for this revision

Independent Reviewer should check only:

1. Does Candidate finalization now have exclusive responsibility for freezing/materializing target-semantic closure before Candidate Revision identity becomes reviewable?
2. Does Snapshot capture only validate an already-finalized immutable Candidate Revision, with no path that mutates C, silently substitutes C', or creates Snapshot-only semantic overrides?
3. Is `CANDIDATE_NOT_REVIEWABLE` (or equivalent) a sufficient failure boundary when current-head resolution finds invalid/legacy/non-finalized Candidate data?
4. Are State and Candidate Snapshot field paths now consistently schema-qualified without changing identity semantics?
5. Did any v4 guarantee regress while making these narrow lifecycle/path corrections?
6. Is any remaining issue promotion-blocking for XCONTRACT-03?

Requested disposition: review this revision independently; do not infer approval from the Design Thread's incorporation of prior feedback.
