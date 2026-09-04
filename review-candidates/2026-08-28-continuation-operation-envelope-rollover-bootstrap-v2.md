# Review Candidate: Continuation Operation Envelope & Rollover Bootstrap Tightening v2

- status: review_candidate
- date: 2026-08-28
- source_role: Designer-2 / Design Thread
- work_item: Issue #3 — Continuation operation envelope & rollover bootstrap tightening
- supersedes_review_target: `836e8319bc7f4164647e081277c443264377e42c` for Issue #3 only
- base_commit: `d8ad8038a89c229421c94fa14d3a0cabad0d105b`
- source_candidate: `review-candidates/2026-08-28-continuation-operation-envelope-rollover-bootstrap-v1.md` @ `836e8319bc7f4164647e081277c443264377e42c`
- committed_dependency: XCONTRACT-03 exact v5 semantics @ `39c7a9b755567792e343bec94a7dd8725c5d37a5`
- scope: ContinuationProposal creation idempotency, immutable execution targets, action-specific State-basis semantics, rollover activation-window quiescence, bootstrap capability invariant, carrier-binding authority bridge, REFRESH stale target semantics, nullable carrier CAS, reviewer-input provenance ownership
- explicitly_out_of_scope: reopening XCONTRACT-03, redesigning Runtime Object / Authority Baseline, broad State Delta + Reducer redesign, Execution Journal schema, full crash recovery algorithm, provider-specific UI automation implementation

---

## 0. Design intent

v2 preserves the v1 architecture and makes one targeted tightening pass after independent review.

The control split remains:

```text
Semantic workflow selection
  -> Control / orchestration reasoning

Canonical Run State mutation
  -> Baseline State Proposal -> Policy -> Authorized Delta -> Reducer

Runtime / carrier continuation
  -> ContinuationProposal -> execution layer
```

`ContinuationProposal` remains a runtime-operation artifact.
It is NOT the Baseline State `Proposal` and never becomes a reducer-bypass authority.

v2 freezes three additional boundaries:

```text
Creation boundary
  same durable decision basis
  -> at most one effective ContinuationProposal

Activation boundary
  final quiescence acquisition
  -> hold predecessor activation fence
  -> revalidate mutable quiescence predicates
  -> authorized carrier CAS apply

Authority boundary
  ContinuationProposal may request/coordinate rollover
  -> canonical carrier binding still changes only through
     State Proposal -> Policy -> Authorized Delta -> Reducer
```

No Execution Journal event/state schema is introduced.

---

# 1. Design handling of independent review findings

These are Design-thread incorporation statuses only, not Review/Integration dispositions.

```yaml
XCONTRACT-R3-01:
  design_handling: INCORPORATED
  change: >
    Rollover now acquires a thread-scoped predecessor activation fence before provisional
    carrier creation, holds it through canonical carrier-CAS apply, and performs a final
    full quiescence revalidation immediately before submitting/using the authorized CAS.
    Any fence break or boundary change aborts activation.

XCONTRACT-R3-02:
  design_handling: INCORPORATED
  change: >
    ContinuationProposal creation now has a durable decision-basis fingerprint and
    idempotent-create uniqueness rule. A crash after proposal persistence but before the
    caller receives proposal_id reuses the already-created proposal instead of allocating
    a second logical operation.

XCONTRACT-R3-03:
  design_handling: INCORPORATED
  change: >
    ROLLOVER activation now explicitly bridges to the committed State mutation pipeline.
    Once provisional carrier identity is known, execution creates or references an immutable
    Baseline State Proposal containing the exact carrier CAS operation; Policy authorizes;
    Reducer ApplyResult.applied is the activation point.

XCONTRACT-R3-04:
  design_handling: INCORPORATED
  change: >
    Expected carrier identity now has one authoritative location only:
    execution_target.expected_current_carrier_id. The duplicate top-level carrier field is removed.

XCONTRACT-R3-05:
  design_handling: INCORPORATED
  change: >
    State basis now has explicit fixed per-action execution semantics:
    CONTINUE_FOCUSED=EXACT, ROLLOVER=EXACT,
    COMPACT=PROVENANCE_ONLY, REFRESH=PROVENANCE_ONLY.

XCONTRACT-R3-06:
  design_handling: INCORPORATED
  change: >
    Canonical carrier-CAS failures use existing Baseline ApplyResult outcomes:
    rejected_stale / rejected_precondition / rejected_invariant.
    Runtime eligibility failures remain separate runtime outcomes.
```

The v1 bootstrap safety, unsupported-adapter fail-closed behavior, exact REFRESH target,
transitive immutable target closure, pending semantic Proposal survival, nullable CAS semantics,
and Review Execution Provenance separation are intentionally preserved.

---

# 2. ContinuationProposal lifecycle

A ContinuationProposal means:

> Perform one exact runtime continuation operation for one Logical Thread, based on one exact durable decision basis, against one immutable correctness-relevant execution target closure.

The lifecycle has two distinct identity boundaries:

```text
Durable Continuation Decision Basis
        ↓ idempotent creation
Immutable ContinuationProposal
        ↓ retry / execution
Runtime action and, where required, separately-authorized canonical mutation
```

The proposal is never reconstructed from live `latest/current` targets during retry.

---

# 3. Durable decision basis and crash-idempotent proposal creation

v1 protected retries of a known `proposal_id`, but did not protect proposal creation when persistence succeeded and the caller crashed before receiving the ID.

v2 adds a durable creation basis.

## 3.1 Decision basis artifact

Before allocating a ContinuationProposal, every input that can change the continuation decision must be represented by a durable immutable basis or by immutable references in that basis.

Conceptually:

```yaml
continuation_decision_basis:
  decision_basis_ref: required
  logical_thread_id: required

  state_target:
    run_id: required
    state_version: required

  finalized_candidate_or_work_ref:
    optional

  active_carrier_id:
    required_nullable: true

  stable_turn_or_checkpoint_ref:
    optional

  runtime_signal_snapshot_ref:
    required_if_runtime_signals_influence_decision: true

  control_or_policy_evaluation_ref:
    required_if_model_or_policy_output_influences_decision: true

  decision_basis_fingerprint: required
  created_at: required
```

This is an artifact-layer decision basis, not new canonical semantic State.

If transient runtime facts influence the decision, the relevant fact set must be captured into an immutable runtime-signal snapshot before proposal creation. The contract does not define the storage schema for that snapshot.

## 3.2 Creation uniqueness invariant

For one Logical Thread:

```text
(logical_thread_id, decision_basis_fingerprint)
```

is the idempotent-create key for an effective ContinuationProposal.

Required invariant:

```text
same logical_thread_id
+ same decision_basis_fingerprint
-> at most one effective ContinuationProposal
```

Creation must behave conceptually as an atomic `create-if-absent` / unique-key insert:

```text
attempt create for basis B
  -> no existing proposal for key
       -> persist CP-17
       -> return CP-17

attempt/retry create for same basis B
  -> CP-17 already exists
       -> return CP-17
       -> do not allocate CP-18
```

This closes:

```text
CP-17 persisted
-> crash before caller receives CP-17 id
-> restart
-> same basis evaluated again
-> reuse CP-17
```

rather than creating a second future Journal idempotency key.

## 3.3 Same basis, different operation result

If the same durable decision basis key somehow maps to a different proposed action/operation fingerprint:

```text
same logical_thread_id
+ same decision_basis_fingerprint
+ different operation_fingerprint
-> CREATION_BASIS_CONFLICT / invariant failure
```

The system must not silently create a second effective proposal under the same basis.

If decision inputs genuinely changed, create a new decision basis with a new fingerprint; then a new ContinuationProposal is allowed.

This rule does not require all evaluators to be globally deterministic. It requires nondeterministic or changed evaluator output to be represented as a changed durable basis/evaluation artifact if it is intended to create a new logical operation.

---

# 4. ContinuationProposal immutable envelope

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

  operation_fingerprint: required
  created_at: required
```

Run State creation-basis identity is:

```text
(run_id, creation_basis.state_version)
```

The duplicated top-level `expected_active_carrier_id` from v1 is removed.

Carrier identity has exactly one authoritative field location:

```text
continuation_proposal.execution_target.expected_current_carrier_id
```

## 4.1 Immutability invariant

After `proposal_id` is persisted:

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

are immutable.

Any correctness-relevant change requires a new decision basis if the decision inputs changed and a new ContinuationProposal identity.

## 4.2 Operation fingerprint

`operation_fingerprint` covers the canonicalized correctness-relevant operation envelope:

```text
hash(
  action
  + run/thread identity
  + decision basis identity
  + state creation basis
  + action-specific immutable execution target closure
)
```

Required integrity rule:

```text
same proposal_id + different operation_fingerprint
-> rejected invariant / ID reuse
```

`proposal_id` remains the logical runtime operation / future Execution Journal correlation key.

## 4.3 Retry rule

Retry of an existing operation:

```text
-> load proposal by proposal_id
-> validate operation_fingerprint
-> validate current facts against frozen preconditions
-> never resolve a replacement latest/current correctness target
```

If the operation is stale/ineligible, the old proposal remains immutable provenance.

A new operation requires a new decision basis/evaluation when relevant and a new proposal ID.

---

# 5. Action-specific State-basis semantics

`creation_basis.state_version` is always required because it records the canonical State basis under which the continuation decision was made.

But its execution meaning is action-specific.

v2 freezes the following v0 rules:

| Action | State basis mode | Execution rule |
|---|---|---|
| CONTINUE_FOCUSED | EXACT | current `(run_id, state_version)` MUST equal creation basis before dispatch |
| COMPACT | PROVENANCE_ONLY | State advance alone does not stale exact-source compaction |
| REFRESH | PROVENANCE_ONLY | State advance alone does not stale exact runtime-surface repair |
| ROLLOVER | EXACT | current `(run_id, state_version)` MUST equal creation basis at canonical activation proposal/apply precondition |

These modes are fixed by action in v0; they are not caller-selectable flags.

## 5.1 EXACT

For `CONTINUE_FOCUSED` and `ROLLOVER`:

```text
current_state(run_id).state_version
!= creation_basis.state_version
-> STALE_STATE_BASIS before external semantic dispatch / activation
```

Reason:

- CONTINUE_FOCUSED request input is frozen from a semantic decision basis; sending it after canonical State changes may execute stale semantic work.
- ROLLOVER projection/checkpoint/bootstrap operation is tied to an exact canonical basis and the carrier CAS itself mutates canonical State.

No same-ID semantic revalidation may rewrite the old proposal onto the new State version.

If continuation is still desired after State advance, new evaluation creates/reuses a proposal from the new durable basis.

## 5.2 PROVENANCE_ONLY

For `COMPACT` and `REFRESH`, State basis records why/when the operation was created but does not by itself create an execution stale condition.

### COMPACT

COMPACT already freezes:

```text
exact carrier
exact checkpoint
exact start/end turn range
exact compaction input
```

An unrelated canonical State change does not alter that exact source artifact operation.

Carrier/source-range/runtime preconditions still apply.

Any State Proposal emitted by the compactor is a separate Baseline State Proposal with its own base-state semantics.

### REFRESH

REFRESH already freezes:

```text
exact Browser Session
exact Adapter Attachment
exact attached carrier
exact Logical Thread active carrier expectation
```

An unrelated canonical semantic State change does not by itself invalidate a browser-surface repair.

If carrier binding changes, the frozen carrier/runtime-target checks stale the operation directly.

---

# 6. Action-specific immutable execution targets

The generic rule remains:

> Any input whose change can change what external/runtime operation will occur is part of ContinuationProposal identity and must be frozen before proposal persistence/retry.

Every action-specific target has one authoritative carrier field:

```text
execution_target.expected_current_carrier_id
```

---

## 6.1 CONTINUE_FOCUSED

```yaml
execution_target:
  kind: CONTINUE_FOCUSED
  expected_current_carrier_id: required non-null
  continuation_input_ref: required immutable/recoverable exact request closure
```

`continuation_input_ref` includes all correctness-relevant request inputs such as exact Context Projection revision, next-intent payload, instruction/template version, and semantically relevant provider request shape.

Execution requires:

```text
State basis mode EXACT passes
active carrier == expected_current_carrier_id
continuation_input_ref integrity/recoverability passes
```

Retry may not compile against latest context.

---

## 6.2 COMPACT

```yaml
execution_target:
  kind: COMPACT
  expected_current_carrier_id: required non-null
  source_checkpoint_id: required
  source_turn_range:
    provider_conversation_id: required
    start_turn_ref: required
    end_turn_ref: required
  compaction_input_ref: required immutable
```

Required consistency at proposal finalization:

```text
source_turn_range.provider_conversation_id
== expected_current_carrier_id
```

or finalization fails as an invalid operation envelope.

State basis mode is PROVENANCE_ONLY.

Retry reads the same exact range and may not include later turns.

---

## 6.3 REFRESH

```yaml
execution_target:
  kind: REFRESH
  expected_current_carrier_id: required non-null
  browser_target:
    browser_session_id: required
    adapter_attachment_id: required
    attached_provider_conversation_id: required
```

Required finalization invariant:

```text
browser_target.attached_provider_conversation_id
== expected_current_carrier_id
```

State basis mode is PROVENANCE_ONLY.

The exact runtime stale behavior from v1 remains unchanged: replacement Browser Session/Attachment is not retargeted under the old proposal ID.

---

## 6.4 ROLLOVER

```yaml
execution_target:
  kind: ROLLOVER
  expected_current_carrier_id: required non-null

  checkpoint_id: required
  predecessor_turn_boundary_ref: required
  context_projection_ref: required immutable/versioned
  bootstrap_input_ref: required immutable
  destination_carrier_spec_ref: required immutable

  activation_strategy:
    RESERVE_THEN_CAS | GUARDED_BOOTSTRAP_THEN_CAS
```

For ROLLOVER:

```text
expected_current_carrier_id
== expected predecessor carrier
```

There is no second `expected_predecessor_id` field.

State basis mode is EXACT.

Retry may not substitute a new checkpoint/projection/destination/bootstrap strategy/current carrier.

---

# 7. Correctness-relevant target closure

All correctness-relevant artifact references must be immutable/versioned/recoverable; runtime lifecycle targets must identify one exact observed lifecycle instance.

Valid examples:

```text
immutable artifact revision ID
content-addressed artifact
artifact_id + immutable revision_id
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

unless they were resolved into exact immutable identities before proposal persistence.

This principle is compatible with committed XCONTRACT-03 but does not modify it.

---

# 8. Rollover activation-window safety

Rollover safe boundary is about execution quiescence, not semantic Proposal settlement.

A durable pending State Proposal may survive rollover.

v2 distinguishes:

```text
preparation-time boundary check
+
activation-window fence
+
final pre-activation revalidation
```

## 8.1 Preparation-time boundary predicates

Before provisional carrier creation starts, all must hold:

```text
1. target Logical Thread is runnable for this operation;
2. active carrier == execution_target.expected_current_carrier_id;
3. predecessor provider generation/streaming is not active;
4. no known unresolved external dispatch ambiguity could still produce an additional
   authoritative predecessor worker result;
5. predecessor_turn_boundary_ref is durably observed/captured;
6. checkpoint_id exists and is tied to the proposal's known State/turn boundary;
7. context_projection_ref is already compiled/frozen;
8. adapter capability for the frozen activation strategy is available;
9. State basis EXACT currently passes.
```

Passing this check does NOT by itself authorize later CAS after a long reserve/bootstrap step.

## 8.2 Predecessor activation fence

Before provisional carrier creation, rollover execution MUST acquire a thread-scoped **predecessor activation fence** and hold it until canonical carrier activation either succeeds or aborts.

Contract capability:

```text
PREDECESSOR_ACTIVATION_FENCE
```

While held, it MUST guarantee for all NOOS-controlled paths:

```text
- no new autonomous/provider dispatch may be started on the predecessor;
- no second carrier-touching operation for the Logical Thread may cross the fence;
- any attempt to dispatch through a fenced NOOS path is rejected/deferred;
- the fence remains associated with the exact logical_thread_id + expected predecessor;
- loss/breakage of the fence is observable and invalidates activation.
```

This is a runtime synchronization capability, not a new canonical State object and not an Execution Journal lifecycle.

### User/manual/external activity

The fence does not pretend NOOS can physically prevent every external actor from using a provider UI.

Therefore autonomous activation additionally requires the adapter/runtime observation boundary to guarantee one of the following for predecessor activity during the fence window:

```text
A. external/user dispatch to the predecessor is prevented/disabled while the activation fence is held;

or

B. any such dispatch/activity synchronously invalidates/breaks the activation fence before canonical activation may be submitted/applied.
```

If the adapter cannot provide either guarantee for the exact execution surface, autonomous rollover is not eligible for that surface at that moment:

```text
-> UNSAFE_ROLLOVER_BOUNDARY / ACTIVATION_FENCE_UNSUPPORTED
-> do not activate provisional carrier
```

Prompt convention is not a substitute.

## 8.3 Final quiescence revalidation

After provisional carrier reservation/bootstrap is complete, but before the carrier-binding State Proposal is submitted/authorized/applied as activation, runtime MUST revalidate all mutable boundary predicates while the activation fence is still held.

At minimum:

```text
1. activation fence is still valid and bound to this thread/predecessor;
2. active carrier is still expected_current_carrier_id;
3. predecessor is still not streaming/generating;
4. there is still no unresolved predecessor dispatch ambiguity;
5. observed predecessor turn boundary is still exactly predecessor_turn_boundary_ref;
6. no newly observed predecessor user/assistant turn exists beyond that frozen boundary;
7. State basis EXACT still matches;
8. provisional carrier remains the exact carrier created for this operation;
9. frozen adapter capability / activation strategy is still valid.
```

Any failure:

```text
-> abort activation
-> do not submit/use a successful carrier-binding apply as this operation's activation
-> provisional carrier remains unbound residue
```

Representative runtime result:

```text
UNSAFE_ROLLOVER_BOUNDARY
```

The same proposal may be retried only if the exact frozen operation is still valid; otherwise a new decision basis/proposal is required.

## 8.4 Fence held through canonical activation

The predecessor activation fence MUST remain held through the canonical carrier-binding apply result.

Conceptually:

```text
acquire fence
-> prepare/reserve/bootstrap provisional Cnew
-> final quiescence revalidation
-> create/reuse exact carrier-binding State Proposal
-> Policy authorization
-> Reducer apply
-> observe ApplyResult
   -> applied: release fence after Cnew is canonically active
   -> rejected_*: release fence; Cnew remains unbound
```

No normal continuation on Cnew is allowed before `ApplyResult.outcome = applied` for the exact carrier-binding delta.

## 8.5 Post-activation predecessor output

After successful canonical activation, the predecessor is no longer the active carrier.

Any later provider output observed from that predecessor:

```text
- remains provenance/runtime evidence;
- MUST NOT be ingested as the current authoritative worker continuation for the Logical Thread;
- does not reverse the carrier CAS;
- may require later reconciliation if external activity occurred outside the guaranteed fence boundary.
```

This clause does not define reconciliation workflow; it only prevents late predecessor output from recreating active-carrier authority.

---

# 9. Pending semantic State Proposals remain decoupled

Rollover does not require pending semantic State Proposals to be reduced/discarded merely to switch carrier.

```text
pending semantic State Proposal remains durable + immutable
carrier rollover has its own canonical CAS mutation
pending proposal later follows existing State Delta stale/revalidation semantics
```

Rollover does not cancel/rewrite/rebase/approve/deny semantic State Proposals.

This preserves semantic proposal lifecycle vs carrier lifecycle separation.

---

# 10. Enforceable pre-activation bootstrap capability invariant

v1 bootstrap safety is preserved.

A provisional new Provider Conversation has no Logical Thread execution authority before canonical active-binding apply succeeds.

Prompt text is not a capability guarantee.

A ROLLOVER proposal freezes one activation strategy.

## 10.1 RESERVE_THEN_CAS

Requires:

```text
CONVERSATION_RESERVATION_WITHOUT_EXECUTION
```

Meaning provider conversation identity can be reserved without a model/worker dispatch.

## 10.2 GUARDED_BOOTSTRAP_THEN_CAS

Requires enforceable:

```text
PREACTIVATION_EXECUTION_SIDE_EFFECTS_DISABLED
```

The preactivation bootstrap must guarantee:

```text
- no external write side effects;
- no persistent-effect tool/connector execution;
- no delegated worker/subagent persistent effects;
- no canonical State mutation path;
- bootstrap response quarantined from normal worker-result ingestion;
- no State Proposal / Human Gate / completion claim generated from quarantined output;
- provisional conversation identity can be discovered and kept unbound until activation.
```

Pure text/model computation may occur only under this restricted capability profile.

## 10.3 Unsupported adapter

If neither reservation-without-execution nor enforceable side-effect-disabled bootstrap is available:

```text
ROLLOVER_BOOTSTRAP_UNSUPPORTED
-> no speculative first prompt
-> active carrier unchanged
```

If the required bootstrap capability or predecessor activation-fence capability disappears before activation, execution fails closed and does not switch strategy under the same proposal ID.

---

# 11. Canonical carrier-binding authority bridge

This section closes the ambiguity between runtime ContinuationProposal and canonical Run State mutation.

## 11.1 ContinuationProposal never calls Reducer as mutation authority

Forbidden architecture:

```text
ContinuationProposal
-> direct canonical Reducer mutation
```

The fact that ContinuationProposal is durable, immutable, or runtime-authorized does not make it a Baseline State Proposal or Authorized Delta.

## 11.2 Activation uses the committed State mutation pipeline

Once the exact provisional `new_provider_conversation_id` exists, ROLLOVER execution creates or references an immutable Baseline **State Proposal** containing exactly one carrier-binding compare-and-set operation.

Conceptually:

```yaml
state_proposal:
  id: SP-...
  run_id: <same run>
  base_state_version: <ContinuationProposal creation_basis.state_version>

  proposer/provenance:
    continuation_proposal_id: CP-...

  operations:
    - op: compare_and_set_active_provider_conversation
      logical_thread_id: <same thread>
      expected_current_provider_conversation_id: <CP execution_target.expected_current_carrier_id>
      new_provider_conversation_id: <exact provisional carrier>
```

Then the existing authority path applies:

```text
immutable State Proposal
-> Policy
-> AuthorizationResult
-> Authorized Delta
-> Reducer
-> ApplyResult
```

Policy MAY automatically authorize this operation under an existing delegated runtime policy, but the Policy step is not skipped.

If policy requires human authority under some deployment configuration, the operation waits/fails according to existing Baseline semantics; Continuation does not bypass it.

## 11.3 Exact bridge invariants

The carrier-binding State Proposal MUST preserve the immutable ContinuationProposal operation identity:

```text
same run_id
same logical_thread_id
same exact base_state_version
same expected_current_carrier_id
exact provisional new_provider_conversation_id created for this ROLLOVER operation
provenance/correlation reference to continuation_proposal_id
```

The bridge MUST NOT:

```text
re-resolve current carrier
change base_state_version
substitute another provisional carrier
compile a new projection/bootstrap target
```

If any such change is needed, the old ROLLOVER operation is stale/ineligible and a new Continuation decision/proposal is required.

## 11.4 Activation point

The exact activation authority point is:

```text
ApplyResult.outcome = applied
for the Authorized Delta carrying the exact carrier-binding CAS
```

Before that:

```text
Cnew = provisional / unbound
```

After that:

```text
Cnew = canonical active carrier
```

A provider conversation ID, URL, bootstrap response, creation timestamp, or most-recently-created conversation never substitutes for this ApplyResult-backed canonical relation.

---

# 12. Carrier CAS semantics and Baseline ApplyResult vocabulary

The carrier-binding operation candidate remains one generic compare-and-set semantic for both initial bind and rollover.

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

`run_id` and `base_state_version` are inherited from the enclosing Baseline State Proposal/Authorized Delta rather than duplicated as an independent authority envelope inside the operation.

Structural preconditions:

```text
current canonical state version == State Proposal base_state_version
current active carrier exactly equals expected_current_provider_conversation_id
```

Initial bind:

```text
expected_current = null
new = C1
```

Rollover:

```text
expected_current = C1
new = C2
```

No separate initial-bind mutation path and no `binding_version` are introduced.

## 12.1 Failure mapping

Canonical apply outcomes use the committed Reducer vocabulary.

```text
current state version != base_state_version
-> ApplyResult.outcome = rejected_stale

current active carrier != expected_current_provider_conversation_id
-> ApplyResult.outcome = rejected_precondition

malformed operation / identity mismatch / impossible envelope invariant
-> ApplyResult.outcome = rejected_invariant
```

On any rejected outcome:

```text
active binding unchanged
no partial rebinding
provisional carrier remains unbound
```

Runtime-level ineligibility results such as:

```text
UNSAFE_ROLLOVER_BOUNDARY
ROLLOVER_BOOTSTRAP_UNSUPPORTED
ADAPTER_CAPABILITY_UNAVAILABLE
RUNTIME_TARGET_STALE
INVALID_OPERATION_FINGERPRINT
CREATION_BASIS_CONFLICT
```

are not new Reducer ApplyResult outcomes.

---

# 13. REFRESH exact target behavior

REFRESH retains v1 semantics.

Before acting, runtime validates the exact frozen:

```text
browser_session_id
adapter_attachment_id
attached_provider_conversation_id
expected_current_carrier_id
```

If the target lifecycle was replaced/superseded:

```text
RUNTIME_TARGET_STALE
-> do not resolve current Browser Session
-> do not refresh replacement target under old proposal_id
```

State advance alone does not stale REFRESH because its state basis mode is PROVENANCE_ONLY.

Carrier change still stales it through exact carrier/runtime-target validation.

---

# 14. ROLLOVER contract-level execution sequence

This ordering is normative at the contract level but does not define Execution Journal event schema.

```text
1. resolve/reuse ContinuationProposal through durable decision-basis idempotent creation
2. load immutable ContinuationProposal by proposal_id
3. validate operation_fingerprint and exact target closure
4. validate ROLLOVER State basis EXACT
5. validate preparation-time execution quiescence predicates
6. acquire predecessor activation fence
7. revalidate fence acquisition did not already invalidate the predecessor boundary
8. validate frozen adapter bootstrap + activation-fence capabilities
9. reserve provisional carrier or perform guarded quarantined bootstrap
10. discover/register exact provisional carrier identity Cnew
11. while fence remains held, perform final full quiescence revalidation
12. create/reuse exact immutable Baseline carrier-binding State Proposal linked to this CP
13. Policy evaluates that State Proposal
14. if authorized, Authorized Delta enters Reducer
15. Reducer returns persisted ApplyResult
    - applied -> Cnew becomes canonical active carrier
    - rejected_stale/precondition/invariant -> Cnew remains unbound
16. release activation fence only after the apply result is known/activation attempt aborts
17. normal continuation on Cnew only after exact ApplyResult.applied
```

No step may retarget the immutable ContinuationProposal to live latest/current identities.

If the fence is lost or final boundary revalidation fails before apply:

```text
-> do not proceed with activation
-> provisional carrier remains unbound
```

---

# 15. Stale and ineligibility semantics

Runtime-level validation outcomes remain factual execution relations, not semantic Review dispositions.

Representative outcomes:

```text
STALE_STATE_BASIS
STALE_ACTIVE_CARRIER
RUNTIME_TARGET_STALE
UNSAFE_ROLLOVER_BOUNDARY
ACTIVATION_FENCE_UNSUPPORTED
ROLLOVER_BOOTSTRAP_UNSUPPORTED
ADAPTER_CAPABILITY_UNAVAILABLE
INVALID_OPERATION_FINGERPRINT
CREATION_BASIS_CONFLICT
```

Canonical carrier-mutation failures use Baseline `ApplyResult.outcome` as defined in §12.1.

Core retry rule:

```text
stale/ineligible
-> never rewrite old proposal onto current/latest target
```

If new facts require a new operation:

```text
new durable decision basis
-> new/reused proposal for that new basis
```

---

# 16. Review Execution Provenance ownership

Committed XCONTRACT-03 remains unchanged.

```text
ReviewSnapshot
-> identifies what was reviewed

Review Execution Provenance
-> records reviewer-specific execution inputs/context
```

Reviewer-specific provenance may include immutable refs to:

```text
review dimension/contract
Context Projection
reviewer prompt/request/input closure
provider/model observation
execution evidence
```

It does not become Candidate Review Target identity merely because it influenced reviewer reasoning.

No new canonical authority is granted by this provenance artifact.

---

# 17. Authority summary

This candidate introduces no new mutation authority.

```text
ContinuationProposal
!= Baseline State Proposal
!= AuthorizationResult
!= Authorized Delta
```

Continuation execution may perform noncanonical runtime/provider actions within allowed capability boundaries.

When ROLLOVER needs to mutate canonical active-carrier binding:

```text
ContinuationProposal
-> Baseline State Proposal (exact carrier CAS)
-> Policy
-> Authorized Delta
-> Reducer
-> ApplyResult.applied
```

This is the only activation-authority bridge frozen by this candidate.

Compaction remains artifact/projection authority.
Review roles remain reasoning/advisory roles.
Bootstrap capability restricts preactivation execution; it does not grant semantic authority.

---

# 18. Counterexamples and required outcomes

## 18.1 Proposal persisted, caller misses ID

```text
basis B42
-> create CP-17 durable
-> crash before caller receives CP-17
-> restart with same basis B42
```

Required:

```text
lookup/create-if-absent by (logical_thread_id, decision_basis_fingerprint)
-> return CP-17
```

Forbidden:

```text
allocate CP-18 for same effective basis
```

## 18.2 Same basis produces different evaluator result

```text
basis B42 -> existing CP-17 / CONTINUE_FOCUSED
retry evaluator on identical durable basis -> proposes ROLLOVER
```

Required:

```text
CREATION_BASIS_CONFLICT
```

or equivalent invariant failure.

A deliberately changed evaluation result must be represented by a new durable evaluation/basis artifact and fingerprint.

## 18.3 Rollover predecessor becomes active again during provisional creation

```text
t0 safe boundary passes on C1
t1 activation fence acquired
t2 provisional C2 creation begins
t3 attempted NOOS dispatch to C1
```

Required:

```text
dispatch rejected/deferred by fence
```

If user/external dispatch is possible, it must either be blocked or synchronously break the fence before activation.

If fence cannot provide that guarantee:

```text
UNSAFE_ROLLOVER_BOUNDARY / ACTIVATION_FENCE_UNSUPPORTED
```

## 18.4 New predecessor turn observed before CAS

Frozen boundary = T10.

Before activation apply, T11 is observed on C1.

Required:

```text
final quiescence revalidation fails
-> no activation
-> C2 remains unbound
```

## 18.5 Late predecessor result after successful activation

C1 -> C2 carrier CAS already applied.
Then C1 produces/returns a late result.

Required:

```text
C1 result may be recorded as provenance/runtime evidence
but cannot be ingested as current authoritative Logical Thread worker continuation
```

No automatic rollback to C1.

## 18.6 ContinuationProposal cannot directly mutate canonical carrier

Forbidden:

```text
CP-20 executor -> Reducer.compare_and_set_active_carrier(...)
```

Required:

```text
CP-20
-> immutable State Proposal SP-9 with exact CAS op
-> Policy
-> Authorized Delta
-> Reducer
-> ApplyResult
```

## 18.7 State advance before CONTINUE_FOCUSED dispatch

CP-21 created at State v50.
Current State becomes v51 before send.

Required:

```text
STALE_STATE_BASIS
-> do not send frozen continuation request
```

## 18.8 State advance before REFRESH

REFRESH targets exact B1/A1/C7 at State v50.
An unrelated semantic update advances State to v51; B1/A1/C7 remain exact/current.

Required:

```text
State advance alone does not stale REFRESH
-> runtime exact-target validation decides eligibility
```

## 18.9 State advance before COMPACT

COMPACT freezes exact C7 + turn range T1..T20 at State v50.
Unrelated semantic State becomes v51, with same carrier/range still available.

Required:

```text
State advance alone does not stale compaction artifact production
```

Any later State mutation proposed from compaction output follows its own Baseline State Proposal basis.

## 18.10 Initial bind and rollover use one State operation semantic

```text
initial:
expected_current = null
new = C1

rollover:
expected_current = C1
new = C2
```

Both use the same carrier-CAS operation under a Baseline State Proposal.

---

# 19. No new durable semantic state / no Execution Journal expansion

This candidate does not create an authoritative Continuation Session or duplicate Run State.

Durable artifacts introduced/reused by the contract are operation/basis/provenance artifacts, not a second semantic state store.

This candidate deliberately does NOT define:

```text
Execution Journal event names
planned/sent/observed/reconciled lifecycle
provider request receipts
crash restart state machine
residue cleanup policy
reconciliation workflow
fence persistence/recovery implementation
provider-specific UI automation APIs
```

The activation fence is a contract-level runtime capability/critical-section invariant only.

The decision-basis idempotent-create key is a ContinuationProposal creation contract, not a Journal event model.

---

# 20. Assumptions

- Runtime Object Model & Authority Model v0.1 remains Baseline.
- State Delta + Reducer Contract v0.1 remains Baseline.
- XCONTRACT-03 remains committed independently.
- Run / Logical Thread are durable NOOS-owned identities.
- one Logical Thread has 0..1 canonical active Provider Conversation binding.
- Provider Conversation may exist while unbound/provisional.
- canonical carrier binding changes only through the Baseline State mutation pipeline.
- Baseline can carry/integrate a narrow carrier compare-and-set operation without changing its authority model.
- Continuation decision inputs that affect proposal creation can be captured as durable immutable basis artifacts/references.
- runtime can enforce thread-scoped single-flight/fence behavior over NOOS-controlled dispatch paths.
- autonomous rollover is fail-closed when external/user predecessor dispatch cannot be prevented or synchronously detected as a fence break.
- Browser Session / Adapter Attachment exact lifecycle identity can be validated.
- bootstrap capabilities remain enforceable runtime/adapter properties, not prompt conventions.

---

# 21. Remaining implementation choices

The following do not change the contract invariants:

- exact schema/storage name for `ContinuationDecisionBasis` artifact;
- exact hash/serialization for decision_basis_fingerprint / operation_fingerprint;
- DB mechanism for unique `logical_thread_id + decision_basis_fingerprint` creation;
- exact runtime implementation of predecessor activation fence;
- exact adapter mechanism for blocking/detecting manual/external predecessor dispatch;
- exact State Delta operation name for carrier compare-and-set;
- whether delegated runtime policy auto-authorizes carrier CAS;
- exact runtime error-code spelling;
- exact artifact location for Review Execution Provenance.

---

# 22. Proposed contract changes relative to v1

v2 proposes only these additional tightenings:

1. Add durable `decision_basis_ref` / `decision_basis_fingerprint` to ContinuationProposal creation.
2. Freeze `logical_thread_id + decision_basis_fingerprint` as the idempotent-create uniqueness key; restart reuses an existing proposal.
3. Treat same-basis/different-operation as an invariant conflict rather than creating duplicate logical operations.
4. Remove duplicated top-level expected-carrier identity; one authoritative `execution_target.expected_current_carrier_id` is used by every action.
5. Freeze per-action State-basis semantics: CONTINUE_FOCUSED/ROLLOVER exact; COMPACT/REFRESH provenance-only.
6. Require predecessor activation fence from final safe-boundary acquisition through canonical carrier apply result.
7. Require adapter behavior that either blocks external/manual predecessor dispatch during the fence or synchronously breaks the fence before activation.
8. Revalidate all mutable quiescence predicates immediately before canonical activation while fence remains held.
9. Explicitly bridge ROLLOVER carrier mutation through Baseline State Proposal -> Policy -> Authorized Delta -> Reducer.
10. Define `ApplyResult.outcome = applied` for the exact carrier-CAS delta as the activation authority point.
11. Reuse committed `rejected_stale / rejected_precondition / rejected_invariant` vocabulary for canonical CAS failures.
12. Preserve v1 bootstrap, REFRESH, pending semantic Proposal, nullable-CAS, immutable target, and Review Provenance semantics unchanged.

---

# 23. Targeted re-review questions

Independent Reviewer should attack only these remaining boundaries:

1. Does the decision-basis uniqueness rule close proposal-creation crash duplication without turning ContinuationDecisionBasis into a second semantic State store?
2. Is the basis closure sufficient to distinguish a genuine new decision from a crash retry of the same logical decision?
3. Does the predecessor activation fence plus final full revalidation close the reserve/bootstrap-to-CAS TOCTOU window at contract level?
4. Is fail-closed behavior strong enough when user/manual predecessor dispatch cannot be blocked or synchronously fence-invalidated?
5. Does late predecessor output after activation remain safely non-authoritative without requiring Execution Journal design here?
6. Is the explicit ContinuationProposal -> Baseline State Proposal -> Policy -> Reducer bridge consistent with committed authority semantics?
7. Does the carrier-CAS State Proposal preserve the exact immutable CP operation identity rather than retargeting live current values?
8. Is one authoritative `execution_target.expected_current_carrier_id` sufficient and internally consistent for all four actions?
9. Are per-action State-basis rules deterministic and conservative enough, especially CONTINUE_FOCUSED=EXACT and REFRESH/COMPACT=PROVENANCE_ONLY?
10. Are canonical CAS failures now cleanly mapped to existing ApplyResult outcomes without inventing another Reducer vocabulary?
11. Did any v1-closed area regress, especially enforceable bootstrap safety, exact REFRESH target, pending semantic Proposal survival, nullable CAS, or Review Execution Provenance separation?
12. Did this revision accidentally enter Execution Journal design or reopen XCONTRACT-03/Runtime Authority Baseline?
13. Is any remaining issue promotion-blocking for Issue #3?

Requested disposition: targeted independent re-review only. Design incorporation does not imply closure.