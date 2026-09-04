# Review Candidate: Continuation Operation Envelope & Rollover Bootstrap Tightening v3

- status: review_candidate
- date: 2026-08-28
- source_role: Designer-2 / Design Thread
- work_item: Issue #3 — Continuation operation envelope & rollover bootstrap tightening
- supersedes_review_target: `731c0d51f29b674d9edfbc1d438b04e231f2b523` for Issue #3 only
- base_commit: `d8ad8038a89c229421c94fa14d3a0cabad0d105b`
- source_candidate: `review-candidates/2026-08-28-continuation-operation-envelope-rollover-bootstrap-v2.md` @ `731c0d51f29b674d9edfbc1d438b04e231f2b523`
- committed_dependency: XCONTRACT-03 exact v5 semantics @ `39c7a9b755567792e343bec94a7dd8725c5d37a5`
- scope: stable ContinuationDecisionBasis identity, crash-idempotent Basis/Proposal creation, Basis→Proposal identity consistency, immutable continuation execution targets, action-specific State-basis semantics, rollover activation-window quiescence, bootstrap capability invariant, carrier-binding authority bridge, first-apply activation gate, REFRESH stale target semantics, nullable carrier CAS, reviewer-input provenance ownership
- explicitly_out_of_scope: reopening XCONTRACT-03, redesigning Runtime Object / Authority Baseline, broad State Delta + Reducer redesign, Execution Journal schema, full crash recovery algorithm, provider-specific UI automation implementation

---

## 0. Design intent

v3 preserves the v2 architecture and makes one narrow durability/identity closure pass after targeted Review #9.

The control split remains unchanged:

```text
Semantic workflow selection
  -> Control / orchestration reasoning

Canonical Run State mutation
  -> Baseline State Proposal -> Policy -> Authorized Delta -> Reducer

Runtime / carrier continuation
  -> ContinuationDecisionBasis -> ContinuationProposal -> execution layer
```

`ContinuationDecisionBasis` and `ContinuationProposal` are artifact-layer runtime-operation inputs. Neither is canonical semantic State and neither is mutation authority.

v3 tightens four boundaries only:

```text
Basis identity boundary
  correctness-relevant semantic basis
  -> canonical semantic fingerprint
  -> crash-idempotent Basis create/reuse

Proposal consistency boundary
  Proposal must exactly agree with its immutable DecisionBasis
  on run/thread/state/carrier identity

First-apply activation boundary
  durable authorization is necessary but not sufficient for a first carrier-CAS apply
  -> current activation fence + post-authorization final quiescence gate
  -> then exact Authorized Delta may enter Reducer

CONTINUE_FOCUSED dispatch boundary
  EXACT State equality is checked at final dispatch authorization immediately before send
  -> no claim of cross-system atomicity with Provider receipt
```

No Execution Journal event/state schema is introduced.

---

# 1. Design handling of Review #9 findings

These are Design-thread incorporation statuses only. They are not Review or Integration dispositions.

```yaml
XCONTRACT-R4-01:
  design_handling: INCORPORATED
  change: >
    decision_basis_fingerprint is now a self-describing canonical fingerprint over
    correctness-relevant semantic basis content, excluding incidental artifact/storage identity.
    ContinuationDecisionBasis itself is crash-idempotently create/reused by
    (logical_thread_id, decision_basis_fingerprint), so equivalent restart capture cannot
    manufacture a second effective basis merely because a new artifact ID was allocated.

XCONTRACT-R4-02:
  design_handling: INCORPORATED
  change: >
    A durable carrier-binding Authorized Delta is not permanently eligible for first apply.
    Every first-apply attempt must pass a current Carrier Activation Apply Gate: no successful
    prior ApplyResult, valid predecessor activation fence, and final full quiescence validation
    performed after authorization/any long wait and immediately before Reducer submission.
    Restart without ApplyResult must reacquire/revalidate; an already-applied delta returns its
    persisted receipt idempotently and does not recreate activation.

XCONTRACT-R4-03:
  design_handling: INCORPORATED
  change: >
    ContinuationProposal finalization now requires exact cross-object equality with its
    DecisionBasis for run_id, logical_thread_id, creation State version, and expected current
    carrier. Mismatch is an invalid operation envelope / creation invariant failure.

XCONTRACT-R4-04:
  design_handling: INCORPORATED
  change: >
    CONTINUE_FOCUSED EXACT semantics now name the final dispatch-authorization check as the
    linearization boundary. The contract does not claim an atomic State-Store + Provider-send
    transaction.
```

All v2-closed areas are intentionally preserved: enforceable bootstrap safety, unsupported-adapter fail closed, exact REFRESH target/no-retarget, pending semantic State Proposal survival, nullable carrier CAS, single expected-carrier field, per-action State-basis modes, Baseline ApplyResult vocabulary, explicit State Proposal→Policy→Reducer bridge, and Review Execution Provenance separation.

---

# 2. Continuation lifecycle and identity layers

The runtime continuation lifecycle is conceptually:

```text
live observations / semantic work basis
        ↓ capture semantic basis
canonicalized correctness-relevant DecisionBasis payload
        ↓ fingerprint + create/reuse
immutable ContinuationDecisionBasis B
        ↓ idempotent proposal creation
immutable ContinuationProposal CP
        ↓ runtime eligibility / execution
external runtime action
        ↓ only when canonical mutation is required
Baseline State Proposal -> Policy -> Authorized Delta -> Reducer
```

Identity layers are distinct:

```text
DecisionBasis semantic identity
  -> decision_basis_fingerprint

DecisionBasis artifact identity
  -> decision_basis_ref

Continuation logical operation identity
  -> proposal_id

Continuation operation payload integrity
  -> operation_fingerprint
```

No layer may be silently replaced by `latest/current` during retry or recovery.

---

# 3. ContinuationDecisionBasis semantic identity

## 3.1 Basis payload

Before ContinuationProposal creation, every input whose difference can legitimately change the continuation decision MUST be represented in one immutable DecisionBasis payload directly or through immutable/recoverable references.

Conceptually:

```yaml
continuation_decision_basis:
  decision_basis_ref: assigned after create/reuse

  logical_thread_id: required

  state_target:
    run_id: required
    state_version: required

  finalized_candidate_or_work_ref: optional immutable/recoverable

  active_carrier_id:
    required: true
    nullable: true

  stable_turn_or_checkpoint_ref: optional immutable/recoverable

  runtime_signal_snapshot_ref:
    required_if_runtime_signals_influence_decision: true

  control_or_policy_evaluation_ref:
    required_if_model_or_policy_output_influences_decision: true

  decision_basis_fingerprint: required self-describing
  created_at: artifact metadata; not semantic unless an explicitly named time value affects decision
```

This artifact is not a second State Store. It freezes the inputs to one continuation decision.

## 3.2 Canonical semantic fingerprint domain

`decision_basis_fingerprint` MUST be computed over a declared canonicalization/hash-domain profile covering the correctness-relevant semantic basis.

Conceptually:

```text
<algorithm>:<basis-canonicalization-profile>:<digest>
```

Example only:

```text
sha256:continuation-basis-canon-v1:<digest>
```

The contract does not require SHA-256 or a particular serialization.

The fingerprint domain MUST include semantic identity/content such as:

```text
logical_thread_id
(run_id, state_version)
active_carrier_id including explicit null
immutable content/revision identity of finalized candidate/work when decision-relevant
immutable content/revision identity of stable turn/checkpoint when decision-relevant
canonical semantic runtime-signal values when decision-relevant
immutable semantic content/revision identity of control/policy evaluation when decision-relevant
other explicitly declared correctness-relevant decision inputs
```

Referenced artifacts enter the basis fingerprint through immutable semantic content/revision identity, not through a newly allocated incidental storage handle alone.

The fingerprint domain MUST exclude incidental representation/transport identity that does not affect the continuation decision, including by default:

```text
decision_basis_ref itself
artifact database row ID
storage path
transport retrieval ID
created_at of the Basis artifact
serialization field ordering after canonicalization
newly allocated runtime-snapshot artifact ID when its canonical semantic snapshot content is equivalent
```

A timestamp/observation time MAY be included when freshness/time is itself a correctness-relevant decision input. In that case it is an explicit semantic field, not incidental `created_at` metadata.

Required rule:

> Equivalent correctness-relevant decision basis content under the same canonicalization profile produces the same `decision_basis_fingerprint`, even if restart recaptures it into a differently allocated physical artifact.

## 3.3 Referenced runtime/evaluation artifacts

If `runtime_signal_snapshot_ref` or `control_or_policy_evaluation_ref` influences the decision, the reference MUST resolve to immutable/recoverable semantic content or immutable revision identity sufficient for deterministic fingerprinting.

This is invalid as sole semantic identity:

```text
runtime_signal_snapshot_ref = RS-42
# where RS-42 is merely a fresh storage ID and equivalent content would become RS-43
```

Valid conceptual approaches include:

```text
content-addressed snapshot
immutable snapshot revision + semantic content hash
canonical inline semantic snapshot payload
immutable evaluation revision + canonical content identity
```

The exact physical storage model is implementation-defined.

---

# 4. Crash-idempotent DecisionBasis create/reuse

The system MUST be able to recover from:

```text
semantic basis captured
-> Basis persisted
-> crash before caller retains decision_basis_ref
```

without creating another effective Basis solely because the caller lost the artifact ID.

## 4.1 Basis uniqueness key

For DecisionBasis creation:

```text
(logical_thread_id, decision_basis_fingerprint)
```

is the idempotent create/reuse key.

Required invariant:

```text
same logical_thread_id
+ same decision_basis_fingerprint
-> at most one effective ContinuationDecisionBasis identity
```

Conceptually:

```text
1. canonicalize semantic basis payload
2. compute decision_basis_fingerprint F
3. create-or-lookup by (logical_thread_id, F)
4. if absent -> persist B42
5. if present -> return existing B42
```

A restart that reconstructs equivalent semantic basis into temporary material MUST recompute the same F and recover B42 rather than persist an effective B43.

Physical duplicate blobs may exist in a low-level store only if orchestration has one authoritative create/reuse mapping and they cannot generate multiple effective Basis identities/ContinuationProposals. Implementations SHOULD enforce a unique key where practical.

## 4.2 Same fingerprint, different semantic payload

If one fingerprint/key is presented with a canonical semantic payload that is not equivalent under the declared profile:

```text
-> BASIS_FINGERPRINT_CONFLICT / rejected invariant
```

It must not overwrite the existing Basis or allocate another effective Basis under the same key.

## 4.3 Profile migration

The canonicalization profile is part of fingerprint identity.

Different profiles are not silently compared as equivalent.

A deliberate profile migration may create a new basis identity even for semantically similar content unless an explicit migration/equivalence policy exists. That is an implementation/governance choice, not a same-profile crash retry.

---

# 5. Crash-idempotent ContinuationProposal creation

After Basis create/reuse, proposal creation retains the v2 invariant.

## 5.1 Proposal uniqueness key

For one Logical Thread:

```text
(logical_thread_id, decision_basis_fingerprint)
```

is also the idempotent-create key for one effective ContinuationProposal derived from that Basis.

Required:

```text
same logical_thread_id
+ same decision_basis_fingerprint
-> at most one effective ContinuationProposal
```

Conceptually:

```text
B42 recovered
-> lookup/create CP by (thread, B42.fingerprint)
-> existing CP-17 => return CP-17
-> absent => persist CP-17
```

This closes both crash windows:

```text
A. B42 persisted -> crash before caller gets B42 -> restart -> recover B42
B. CP-17 persisted -> crash before caller gets CP-17 -> restart -> recover CP-17
```

## 5.2 Same Basis, different operation result

If the same Basis key would produce a different action/operation fingerprint:

```text
same thread
+ same decision_basis_fingerprint
+ different operation_fingerprint
-> CREATION_BASIS_CONFLICT / invariant failure
```

No CP-18 is created as another effective logical operation.

If evaluator output legitimately changes the intended operation, the decision input/evaluation artifact must be represented as a changed semantic Basis, producing a new fingerprint.

---

# 6. ContinuationProposal immutable envelope

Minimum envelope:

```yaml
continuation_proposal:
  proposal_id: required
  run_id: required
  logical_thread_id: required

  decision_basis_ref: required
  decision_basis_fingerprint: required

  action:
    CONTINUE_FOCUSED | COMPACT | REFRESH | ROLLOVER

  creation_basis:
    state_version: required

  execution_target:
    # action-specific immutable tagged union
    expected_current_carrier_id:
      required: true
      nullable: action-specific

  operation_fingerprint: required
  created_at: required
```

Carrier identity has one authoritative Proposal field location only:

```text
continuation_proposal.execution_target.expected_current_carrier_id
```

No duplicate top-level `expected_active_carrier_id` or ROLLOVER-only `expected_predecessor_id` exists.

## 6.1 Basis → Proposal equality invariants

Proposal finalization MUST load the immutable DecisionBasis referenced by `decision_basis_ref`, verify its semantic fingerprint, and enforce all of the following before `proposal_id` becomes effective:

```text
proposal.decision_basis_fingerprint
== basis.decision_basis_fingerprint

proposal.logical_thread_id
== basis.logical_thread_id

proposal.run_id
== basis.state_target.run_id

proposal.creation_basis.state_version
== basis.state_target.state_version

proposal.execution_target.expected_current_carrier_id
== basis.active_carrier_id
```

For the four carrier-touching actions in this Candidate, an action requiring a non-null active carrier MUST reject a Basis whose `active_carrier_id = null` rather than inventing a carrier during Proposal finalization.

Initial carrier binding remains supported by the generic canonical CAS operation with `expected_current = null`; it is not produced by silently violating a carrier-touching action's declared target schema.

Any mismatch above is:

```text
INVALID_OPERATION_ENVELOPE / creation invariant failure
```

The Proposal is not finalized and cannot execute.

These equality rules are additional to action-specific internal consistency checks such as COMPACT source carrier or REFRESH attachment carrier equality.

## 6.2 Immutability

After `proposal_id` is persisted, all correctness-relevant fields are immutable:

```text
proposal_id
run_id
logical_thread_id
decision_basis_ref
decision_basis_fingerprint
action
creation_basis
execution_target
operation_fingerprint
```

Any semantic change requires a new Basis where decision inputs changed and a new Proposal identity.

## 6.3 Operation fingerprint

`operation_fingerprint` covers canonicalized correctness-relevant Proposal semantics:

```text
hash(
  action
  + run/thread identity
  + decision basis semantic identity
  + State creation basis
  + action-specific immutable execution target closure
)
```

Required integrity rule:

```text
same proposal_id + different operation_fingerprint
-> rejected invariant / ID reuse
```

`proposal_id` remains the logical runtime operation / future Journal correlation key.

## 6.4 Retry rule

Retry:

```text
-> load exact proposal by proposal_id
-> validate operation fingerprint and Basis consistency
-> validate current facts against frozen preconditions
-> never resolve replacement latest/current correctness targets
```

Stale/ineligible Proposal remains immutable provenance.

---

# 7. Action-specific State-basis semantics

`creation_basis.state_version` is always recorded because it is part of the DecisionBasis.

v3 preserves v2 modes:

| Action | State basis mode | Execution rule |
|---|---|---|
| CONTINUE_FOCUSED | EXACT | equality required at final dispatch-authorization check immediately preceding external send |
| COMPACT | PROVENANCE_ONLY | State advance alone does not stale exact-source compaction |
| REFRESH | PROVENANCE_ONLY | State advance alone does not stale exact runtime-surface repair |
| ROLLOVER | EXACT | equality required for activation preparation and again at first-apply activation gate / canonical CAS basis |

These modes are fixed by action in v0.

## 7.1 CONTINUE_FOCUSED EXACT linearization boundary

Before external semantic send, runtime MUST perform the final dispatch-authorization validation:

```text
current_state(run_id).state_version
== proposal.creation_basis.state_version

and

current active carrier
== proposal.execution_target.expected_current_carrier_id

and

continuation_input_ref integrity/recoverability passes
```

The successful completion of this final validation is the contract's dispatch-authorization / linearization boundary.

It MUST occur immediately before handing the exact frozen request to the provider adapter/send path.

The contract does NOT claim:

```text
State Store version check
+ Provider network receipt
```

form one cross-system atomic transaction.

Therefore a State change that occurs after successful final dispatch authorization does not retroactively make the already-authorized send a protocol violation. If stronger cross-system serializability is later required, that needs a separate dispatch fence/lease design.

If State mismatch is observed at the final authorization check:

```text
STALE_STATE_BASIS
-> do not send
```

## 7.2 ROLLOVER EXACT

ROLLOVER State equality is required at preparation and at the first-apply activation gate.

The immutable carrier-binding State Proposal/Authorized Delta retains the same base_state_version.

Any mismatch before first apply prevents activation under this Proposal.

## 7.3 COMPACT / REFRESH PROVENANCE_ONLY

Unrelated State advance alone does not stale these exact-source/exact-runtime maintenance operations.

Their frozen source/carrier/runtime target checks still apply.

Any State Proposal later emitted from compaction output has its own Baseline base-state semantics.

---

# 8. Action-specific immutable execution targets

Any input whose change can change what external/runtime operation occurs is part of Proposal identity and is frozen before persistence.

## 8.1 CONTINUE_FOCUSED

```yaml
execution_target:
  kind: CONTINUE_FOCUSED
  expected_current_carrier_id: required non-null
  continuation_input_ref: required immutable/recoverable exact request closure
```

Retry never compiles against latest context.

## 8.2 COMPACT

```yaml
execution_target:
  kind: COMPACT
  expected_current_carrier_id: required non-null
  source_checkpoint_id: required
  source_turn_range:
    provider_conversation_id: required
    start_turn_ref: required
    end_turn_ref: required
  compaction_input_ref: required immutable/recoverable
```

Finalization requires:

```text
source_turn_range.provider_conversation_id
== expected_current_carrier_id
```

Retry reads the same exact range and does not include later turns.

## 8.3 REFRESH

```yaml
execution_target:
  kind: REFRESH
  expected_current_carrier_id: required non-null
  browser_target:
    browser_session_id: required
    adapter_attachment_id: required
    attached_provider_conversation_id: required
```

Finalization requires:

```text
browser_target.attached_provider_conversation_id
== expected_current_carrier_id
```

Replacement Browser Session/Attachment is never retargeted under the old Proposal.

## 8.4 ROLLOVER

```yaml
execution_target:
  kind: ROLLOVER
  expected_current_carrier_id: required non-null
  checkpoint_id: required
  predecessor_turn_boundary_ref: required
  context_projection_ref: required immutable/versioned
  bootstrap_input_ref: required immutable/recoverable
  destination_carrier_spec_ref: required immutable/recoverable
  activation_strategy:
    RESERVE_THEN_CAS | GUARDED_BOOTSTRAP_THEN_CAS
```

`expected_current_carrier_id` is the predecessor. There is no second predecessor identity field.

Retry cannot substitute latest checkpoint/projection/destination/bootstrap strategy/current carrier.

---

# 9. Immutable correctness target closure

All correctness-relevant artifact references must resolve to immutable/versioned/recoverable content. Runtime lifecycle targets must identify one exact observed lifecycle instance.

Valid conceptual forms:

```text
immutable artifact revision ID
content-addressed artifact
artifact ID + immutable revision/content hash backed by recoverable storage
exact TurnRef identity/fingerprint
exact BrowserSession / AdapterAttachment lifecycle identity
```

Forbidden unresolved selectors include:

```text
latest checkpoint
current projection
current browser session
active tab
latest prompt template
current carrier
latest turn
```

unless resolved and frozen before Basis/Proposal finalization.

---

# 10. Rollover execution quiescence and activation fence

Rollover safe boundary remains execution-oriented. Durable pending semantic State Proposals may survive carrier rollover.

## 10.1 Preparation predicates

Before provisional carrier creation:

```text
1. target Logical Thread runnable;
2. active carrier == expected_current_carrier_id;
3. predecessor not streaming/generating;
4. no known unresolved dispatch ambiguity;
5. predecessor turn boundary exactly captured;
6. checkpoint exists and is tied to known State/turn boundary;
7. Context Projection already frozen;
8. activation strategy capability available;
9. State basis EXACT passes.
```

## 10.2 PREDECESSOR_ACTIVATION_FENCE

Before provisional carrier creation, autonomous ROLLOVER MUST acquire a thread-scoped predecessor activation fence.

For NOOS-controlled paths it prevents new predecessor dispatch and competing carrier-touching operations.

Fence loss/breakage is observable and invalidates activation.

For manual/external predecessor activity, the adapter/runtime must provide an enforceable guarantee that while the critical activation window is open, such activity is either:

```text
A. prevented/disabled;

or

B. synchronously invalidates the fence before a first-apply activation attempt may enter Reducer.
```

If neither is enforceable for the exact surface:

```text
ACTIVATION_FENCE_UNSUPPORTED / UNSAFE_ROLLOVER_BOUNDARY
-> fail closed
```

Prompt wording is not a safety mechanism.

## 10.3 Provisional carrier creation

With a valid fence, execution follows the frozen strategy:

```text
RESERVE_THEN_CAS
or
GUARDED_BOOTSTRAP_THEN_CAS
```

A provisional carrier has no Logical Thread execution authority before canonical activation.

Guarded preactivation bootstrap remains side-effect-disabled and quarantined from semantic ingestion.

---

# 11. Canonical carrier-binding authority bridge

`ContinuationProposal` cannot call Reducer directly as mutation authority.

Once exact provisional `Cnew` exists, execution creates/reuses an immutable Baseline State Proposal containing exactly the carrier compare-and-set operation:

```yaml
state_proposal:
  run_id: <CP.run_id>
  base_state_version: <CP.creation_basis.state_version>
  proposer/provenance:
    continuation_proposal_id: <CP.proposal_id>
  operations:
    - op: compare_and_set_active_provider_conversation
      logical_thread_id: <CP.logical_thread_id>
      expected_current_provider_conversation_id: <CP.execution_target.expected_current_carrier_id>
      new_provider_conversation_id: <exact provisional Cnew>
```

Then:

```text
State Proposal
-> Policy
-> AuthorizationResult
-> Authorized Delta
-> Carrier Activation Apply Gate
-> Reducer
-> ApplyResult
```

The State Proposal/Authorized Delta MUST preserve the exact CP run/thread/base/current-carrier/new-carrier identity and must not retarget live values.

Policy may automatically authorize under delegated policy, but Policy is not skipped.

Authorization is mutation permission. It is not permanent proof that runtime quiescence still holds.

---

# 12. Durable authorization does not imply permanent first-apply eligibility

This section is the v3 first-apply closure.

A carrier-binding Authorized Delta is durable according to the Baseline.

Required invariant:

> `Authorized Delta exists` means the exact canonical mutation has authority if its current apply eligibility also holds. It does NOT mean the carrier-CAS may be first-applied later without a current predecessor activation fence and fresh quiescence validation.

## 12.1 Distinguish replay of an already-applied Delta from first apply

Before any activation execution attempts a carrier-CAS apply, the system MUST determine whether the exact Authorized Delta already has a successful persisted transition/ApplyResult under Baseline idempotency semantics.

Conceptually:

```text
successful persisted ApplyResult exists for exact delta_id + fingerprint
-> return/use original persisted ApplyResult
-> do not perform another activation
-> no new first-apply quiescence decision is needed
```

This is receipt replay, not a new activation.

If no successful ApplyResult exists, the next Reducer invocation could cause the **first apply** and therefore MUST pass §12.2.

## 12.2 Carrier Activation Apply Gate

Every code path capable of causing first apply of the carrier-binding Authorized Delta MUST pass one contract-level runtime gate.

Conceptual name:

```text
Carrier Activation Apply Gate
```

The name/storage implementation is not normative. The invariant is.

Immediately before the exact Authorized Delta is submitted to Reducer for a possible first apply, the gate MUST establish:

```text
1. no successful ApplyResult already exists for this exact delta;
2. exact ContinuationProposal / State Proposal / Authorized Delta identity chain validates;
3. a current PREDECESSOR_ACTIVATION_FENCE is held for this thread + expected predecessor;
4. fence capability is still enforceable on the exact runtime surface;
5. active carrier still equals expected_current_carrier_id;
6. predecessor still not streaming/generating;
7. no unresolved predecessor dispatch ambiguity exists;
8. observed predecessor turn boundary still exactly equals predecessor_turn_boundary_ref;
9. no new predecessor user/assistant turn has been observed beyond the frozen boundary;
10. ROLLOVER State basis EXACT still equals Authorized Delta base_state_version;
11. provisional Cnew is still the exact new carrier named by the State Proposal/Authorized Delta;
12. bootstrap/adapter capability assumptions for this operation remain valid.
```

Only after all checks pass may the unchanged exact Authorized Delta enter Reducer for possible first apply.

This final quiescence validation MUST occur **after Policy authorization has completed and after any potentially long authorization/Human-Gate wait**, not only before authorization.

It must be the last runtime eligibility gate immediately preceding Reducer submission.

## 12.3 Authorization wait / Human Gate

If Policy does not authorize immediately, the system MUST NOT assume a previously acquired fence/quiescence observation remains valid indefinitely.

A conforming implementation may:

```text
A. keep a valid enforceable fence through the wait, then still revalidate after authorization;

or

B. release/lose the earlier fence while waiting, then after authorization reacquire the exact predecessor fence and rerun the full final quiescence validation before first apply.
```

If reacquisition/final validation fails:

```text
-> Authorized Delta remains durable provenance
-> do not first-apply it
-> provisional carrier remains unbound
-> runtime reports UNSAFE_ROLLOVER_BOUNDARY / equivalent ineligibility
```

Authorization is not revoked or rewritten by Continuation; the exact mutation simply fails current runtime first-apply eligibility.

## 12.4 Crash/restart after authorization but before first apply

Counterexample to close:

```text
fence valid
-> State Proposal authorized
-> durable Authorized Delta SD-9 exists
-> crash
-> no successful ApplyResult exists
```

Required restart behavior:

```text
load exact CP / State Proposal / Authorized Delta
-> check no successful ApplyResult exists
-> reacquire valid predecessor activation fence
-> rerun full final quiescence validation against frozen operation
-> if all pass: submit exact SD-9 for first apply
-> else: do not apply SD-9
```

Forbidden:

```text
restart
-> find durable Authorized Delta
-> call Reducer directly because authorization already exists
```

## 12.5 Apply-path conformance

Any implementation path that can cause **first apply** of this carrier-binding operation while bypassing the Carrier Activation Apply Gate is nonconforming to this contract.

The gate does not modify the Authorized Delta, does not reauthorize it, and does not become canonical mutation authority. It only decides whether the already-authorized runtime-sensitive mutation is currently eligible to be submitted for first apply.

If a deployment architecture cannot guarantee that all first-apply calls for this operation pass the gate, then v0 autonomous rollover on that architecture is not eligible unless a stronger Reducer-verifiable durable activation guard is introduced by a later design.

v3 does not introduce that heavier token.

## 12.6 Fence lifetime through ApplyResult

Once the first-apply gate passes and Reducer submission begins, the predecessor activation fence remains held until the canonical ApplyResult is known or the attempt aborts.

The adapter/runtime must continue enforcing the declared predecessor-activity guarantee during this critical section.

If it cannot prevent or synchronously gate predecessor activity through this interval, autonomous rollover is fail-closed for that execution surface.

---

# 13. Activation point and ApplyResult semantics

The sole canonical activation point remains:

```text
ApplyResult.outcome = applied
for the exact Authorized Delta carrying the exact carrier CAS
```

Before that:

```text
Cnew = provisional / unbound
```

After that:

```text
Cnew = canonical active carrier
```

Already-successful retry:

```text
same delta_id + same fingerprint + successful transition exists
-> return original ApplyResult
-> do not re-execute carrier mutation
```

Canonical failure vocabulary remains Baseline:

```text
state version mismatch -> rejected_stale
expected current carrier mismatch -> rejected_precondition
malformed/identity invariant mismatch -> rejected_invariant
```

Runtime eligibility outcomes such as `UNSAFE_ROLLOVER_BOUNDARY` are not new Reducer ApplyResult values.

---

# 14. Generic active-carrier CAS with nullable expected current

The canonical operation semantic remains:

```yaml
op: compare_and_set_active_provider_conversation
logical_thread_id: required
expected_current_provider_conversation_id:
  required: true
  nullable: true
new_provider_conversation_id:
  required: true
  nullable: false
```

Enclosing State Proposal/Authorized Delta carries `run_id` and `base_state_version`.

Initial bind:

```text
null -> C1
```

Rollover:

```text
C1 -> C2
```

No second initial-bind mutation path and no `binding_version` are introduced.

---

# 15. Bootstrap capability boundary preserved

A provisional new Provider Conversation has no Logical Thread execution authority before canonical activation.

Allowed frozen strategies:

```text
RESERVE_THEN_CAS
  requires CONVERSATION_RESERVATION_WITHOUT_EXECUTION

GUARDED_BOOTSTRAP_THEN_CAS
  requires PREACTIVATION_EXECUTION_SIDE_EFFECTS_DISABLED
```

Guarded bootstrap MUST guarantee no persistent external effects, no canonical State mutation, no normal worker-result ingestion, and no State Proposal/Human Gate/completion claim generated from quarantined bootstrap output.

Prompt-only `do nothing` instructions are insufficient.

If neither strategy is enforceable:

```text
ROLLOVER_BOOTSTRAP_UNSUPPORTED
-> no speculative first prompt
-> active carrier unchanged
```

If required bootstrap/fence capability disappears before first apply, activation fails closed.

---

# 16. REFRESH exact-target behavior preserved

REFRESH targets one exact Browser Session / Adapter Attachment / attached carrier lifecycle tuple.

Before acting, runtime validates frozen:

```text
browser_session_id
adapter_attachment_id
attached_provider_conversation_id
expected_current_carrier_id
```

Replacement/supersession:

```text
RUNTIME_TARGET_STALE
-> do not resolve current Browser Session
-> do not refresh replacement target under old proposal_id
```

State advance alone does not stale REFRESH because its basis mode is PROVENANCE_ONLY.

---

# 17. Pending semantic State Proposals remain decoupled

Rollover does not require unrelated pending semantic State Proposals to be reduced/discarded merely to switch carrier.

```text
pending semantic State Proposal remains durable + immutable
carrier rollover has its own exact State Proposal/CAS
pending semantic proposal later follows normal Baseline stale/revalidation semantics
```

Continuation does not cancel/rewrite/rebase/approve/deny those proposals.

---

# 18. Late predecessor output

After successful activation, predecessor output observed later:

```text
- remains provenance/runtime evidence;
- MUST NOT become current authoritative Logical Thread worker continuation;
- does not reverse carrier CAS;
- may require later reconciliation outside this contract.
```

This does not define Execution Journal/reconciliation workflow.

---

# 19. Review Execution Provenance remains separate

Committed XCONTRACT-03 is unchanged.

```text
ReviewSnapshot
-> what was reviewed

Review Execution Provenance
-> reviewer-specific execution inputs/context
```

Reviewer-specific Context Projection, prompt/request closure, review dimension, model/provider observation, and execution evidence remain provenance and do not become Candidate Review Target identity merely because they influenced reasoning.

---

# 20. Normative ROLLOVER execution ordering

The contract-level order is:

```text
1. canonicalize semantic DecisionBasis payload
2. compute self-describing decision_basis_fingerprint
3. create/reuse DecisionBasis by (logical_thread_id, fingerprint)
4. create/reuse ContinuationProposal by same durable basis key
5. validate Basis→Proposal equality + operation_fingerprint + immutable target closure
6. validate ROLLOVER preparation predicates / State EXACT
7. acquire predecessor activation fence
8. reserve provisional carrier or guarded-bootstrap exact Cnew
9. create/reuse exact carrier-binding Baseline State Proposal linked to CP
10. Policy evaluates State Proposal
11. if not yet authorized, wait according to Baseline policy; no first apply occurs
12. once exact Authorized Delta exists, check whether successful ApplyResult already exists
    - yes -> use persisted receipt; do not recreate activation
    - no -> possible first apply path continues
13. acquire/reconfirm current predecessor activation fence as needed
14. AFTER authorization/any wait, perform full final quiescence validation
15. pass Carrier Activation Apply Gate
16. submit unchanged exact Authorized Delta to Reducer while fence remains held
17. observe persisted ApplyResult
    - applied -> Cnew becomes canonical active carrier
    - rejected_stale/precondition/invariant -> Cnew remains unbound
18. release fence after result/abort
19. normal continuation on Cnew only after exact ApplyResult.applied
```

No step may retarget Basis/Proposal/State Proposal/Authorized Delta to current/latest values.

---

# 21. Counterexamples and required outcomes

## 21.1 Equivalent runtime snapshot recaptured after crash

Before crash:

```text
semantic runtime facts = X
physical snapshot artifact = RS-41
basis fingerprint = F7
Basis B42 persisted
```

Restart recaptures equivalent facts:

```text
semantic runtime facts = X
physical snapshot artifact = RS-42
```

Required:

```text
canonical semantic fingerprint still = F7
create/reuse (thread, F7)
-> return B42
-> proposal create/reuse returns existing CP-17 if already created
```

Forbidden:

```text
RS-41 vs RS-42 incidental IDs alone
-> F7 vs F8
-> duplicate effective Basis/Proposal
```

## 21.2 Basis persisted, caller loses ID

```text
B42 persisted
-> crash before caller retains B42.ref
-> restart reconstructs equivalent semantic basis
```

Required:

```text
recompute same fingerprint
-> lookup/create-reuse B42
```

## 21.3 Basis/Proposal identity mismatch

```text
Basis:
  run=R1 thread=T1 state=v50 carrier=C1

Proposal tries:
  run=R1 thread=T1 state=v50 carrier=C2
```

Required:

```text
INVALID_OPERATION_ENVELOPE
-> no effective Proposal
```

Same for run/thread/state mismatch.

## 21.4 Authorized Delta survives crash but was never applied

```text
fence acquired
-> State Proposal authorized
-> SD-9 durable
-> crash
-> no successful ApplyResult
-> predecessor activity occurs
```

Required restart:

```text
SD-9 authorization alone is insufficient
-> reacquire fence
-> final post-authorization quiescence validation
-> predecessor boundary changed => fail runtime eligibility
-> do not first-apply SD-9
```

## 21.5 Authorized Delta already applied before receipt loss

```text
SD-9 applied
-> State/ApplyResult/Transition committed atomically
-> caller crashes before receipt
```

Required:

```text
restart sees successful persisted transition
-> return original ApplyResult
-> no new activation fence needed to recreate an already-completed activation
-> no second mutation
```

## 21.6 Human authorization wait

```text
provisional C2 exists
-> State Proposal waits at Human Gate
-> predecessor remains C1 and later receives new turn T11
-> Human approves
```

Required:

```text
Authorized Delta may now exist
but first-apply gate revalidates after authorization
-> frozen boundary was T10, observed boundary now T11
-> UNSAFE_ROLLOVER_BOUNDARY
-> do not apply carrier CAS
```

## 21.7 CONTINUE_FOCUSED State change after dispatch authorization

```text
final check passes at v50
-> exact send handed to provider adapter
-> concurrent unrelated State becomes v51
```

Contract interpretation:

```text
send was authorized at final dispatch boundary
no cross-system atomic State/send transaction is claimed
```

A v51 observed before the final check would have produced `STALE_STATE_BASIS` and no send.

---

# 22. Authority summary

No new canonical authority is introduced.

```text
ContinuationDecisionBasis
!= canonical State

ContinuationProposal
!= Baseline State Proposal
!= AuthorizationResult
!= Authorized Delta

Carrier Activation Apply Gate
!= Policy
!= Reducer
```

The Apply Gate only enforces runtime eligibility for first submission of an already-authorized runtime-sensitive canonical mutation.

Canonical mutation still follows:

```text
State Proposal -> Policy -> Authorized Delta -> Reducer -> ApplyResult
```

The gate cannot rewrite the Authorized Delta or bypass Policy.

---

# 23. No Execution Journal expansion

This Candidate does NOT define:

```text
planned/sent/observed/reconciled event schema
provider request receipt schema
full crash-restart state machine
residue cleanup policy
reconciliation workflow
fence persistence schema
provider-specific UI automation APIs
```

The restart rules in §12/§21 are contract eligibility invariants needed to prevent unsafe first apply. They do not define a Journal lifecycle.

---

# 24. Assumptions

- Runtime Object Model & Authority Model v0.1 remains Baseline.
- State Delta + Reducer Contract v0.1 remains Baseline.
- XCONTRACT-03 remains committed independently.
- Run / Logical Thread are durable NOOS-owned identities.
- one Logical Thread has 0..1 canonical active Provider Conversation binding.
- Provider Conversation may exist while unbound/provisional.
- canonical carrier binding changes only through Baseline mutation authority.
- DecisionBasis correctness-relevant inputs can be represented through immutable/recoverable semantic content identity.
- a stable declared canonicalization profile can compute decision_basis_fingerprint.
- Basis and Proposal create/reuse can enforce effective uniqueness by thread + fingerprint.
- all first-apply paths for carrier-binding Authorized Delta can be routed through the runtime activation gate; otherwise autonomous rollover is fail-closed for that architecture.
- runtime can enforce predecessor activation fence over NOOS-controlled dispatch and the declared external/manual-activity guarantee through the first-apply critical section.
- Browser Session / Adapter Attachment exact lifecycle identity can be validated.
- bootstrap capabilities are enforceable runtime/adapter properties, not prompt conventions.

---

# 25. Remaining implementation choices

These choices do not change the contract invariants:

- exact hash algorithm and serialization for `decision_basis_fingerprint` / `operation_fingerprint`;
- exact name/version syntax for basis canonicalization profile;
- physical storage model for runtime-signal/evaluation snapshots;
- DB/index implementation for Basis and Proposal create/reuse uniqueness;
- exact artifact schema/name for ContinuationDecisionBasis;
- exact runtime implementation of predecessor activation fence;
- exact runtime API/name for Carrier Activation Apply Gate;
- exact adapter mechanism for preventing/synchronously gating manual predecessor activity;
- exact State Delta carrier-CAS operation name;
- whether delegated policy auto-authorizes carrier CAS;
- exact runtime error-code spelling;
- exact storage location for Review Execution Provenance.

---

# 26. Proposed contract changes relative to v2

v3 adds only these narrow tightenings:

1. Define `decision_basis_fingerprint` over canonical correctness-relevant semantic basis, not incidental artifact identity.
2. Make the basis fingerprint self-describing by algorithm + canonicalization/hash-domain profile.
3. Require immutable semantic content/revision identity for referenced runtime/evaluation inputs used in the fingerprint.
4. Add crash-idempotent ContinuationDecisionBasis create/reuse by `(logical_thread_id, decision_basis_fingerprint)`.
5. Preserve Proposal create/reuse on the same stable basis key, closing both Basis and Proposal caller-missed-ID crash windows.
6. Add Basis→Proposal equality invariants for run/thread/state/current-carrier identity.
7. Clarify CONTINUE_FOCUSED EXACT at the final dispatch-authorization boundary without claiming cross-system State/send atomicity.
8. Make durable carrier-CAS authorization insufficient by itself for first apply.
9. Require every possible first apply to pass current activation fence + post-authorization final quiescence validation immediately before Reducer submission.
10. Require restart with Authorized Delta but no ApplyResult to reacquire/revalidate before first apply.
11. Preserve Baseline idempotent receipt replay for an already-successful ApplyResult without recreating activation.
12. Preserve all v2-closed bootstrap, REFRESH, pending semantic Proposal, nullable-CAS, authority, and provenance boundaries.

---

# 27. Targeted re-review questions

Independent Reviewer should attack only:

1. Does semantic canonicalization of DecisionBasis prevent equivalent restart recapture from producing a new basis fingerprint solely because artifact IDs/timestamps changed?
2. Does Basis create/reuse by `(logical_thread_id, fingerprint)` close the earlier caller-missed-Basis-ID crash window without creating a second State Store?
3. Are fingerprint inclusion/exclusion rules precise enough while leaving physical serialization/hash implementation open?
4. Do Basis→Proposal equality invariants completely eliminate run/thread/state/carrier cross-object ambiguity?
5. Can a durable Authorized carrier-CAS Delta ever be first-applied after fence loss or a long authorization wait without passing a current activation fence + fresh final quiescence gate?
6. After crash with Authorized Delta but no ApplyResult, is reacquire/revalidate-before-first-apply mandatory and sufficient at contract level?
7. Does already-successful ApplyResult replay remain compatible with Baseline idempotency without demanding a new activation fence for a completed mutation?
8. Is the requirement that all first-apply paths pass the activation gate enforceable as an architecture invariant without prematurely designing a durable guard token?
9. Is CONTINUE_FOCUSED EXACT linearization wording clear that State equality is required immediately before send authorization, but no cross-system atomicity is claimed?
10. Did any v2-closed area regress?
11. Did v3 accidentally introduce a new mutation authority, semantic State object, or Execution Journal schema?
12. Is any remaining issue promotion-blocking for Issue #3?

Requested disposition: targeted independent re-review only. Design incorporation does not imply closure.
