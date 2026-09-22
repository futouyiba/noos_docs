# Review Candidate: Continuation Operation Envelope & Rollover Bootstrap Tightening v1

- status: review_candidate
- date: 2026-08-28
- source_role: Designer-2 / Design Thread
- work_item: Issue #3 — Continuation operation envelope & rollover bootstrap tightening
- base_commit: `d8ad8038a89c229421c94fa14d3a0cabad0d105b`
- source_candidate: `review-candidates/2026-08-28-continuation-role-cross-contract-v2.md` @ `856837acd93e61d99da9882d2f15ea3cc6c678f2`
- committed_dependency: XCONTRACT-03 exact v5 semantics @ `39c7a9b755567792e343bec94a7dd8725c5d37a5`
- scope: continuation operation identity, action-specific immutable execution targets, rollover safe boundary, bootstrap capability invariant, REFRESH stale target semantics, carrier-binding CAS null predecessor, reviewer-input provenance ownership
- explicitly_out_of_scope: reopening XCONTRACT-03, redesigning Runtime Object / Authority Baseline, redesigning State Delta + Reducer Baseline, Execution Journal schema, full crash recovery algorithm, provider-specific UI automation details

---

## 0. Design intent

This revision is a narrow tightening of the previous Continuation × Multi-Conversation candidate.

It does not introduce a new runtime subsystem and does not create a second durable semantic state machine.

The existing control split remains:

```text
Semantic workflow selection
  -> Control / orchestration reasoning

Canonical Run State mutation
  -> State Proposal -> Policy -> Reducer

Runtime / carrier continuation
  -> ContinuationProposal -> execution layer
```

`ContinuationProposal` in this document is a runtime-operation artifact. It is not the Baseline State `Proposal` and does not obtain canonical mutation authority merely because it is durable and immutable.

The purpose of this revision is to make one ContinuationProposal mean one exact, retryable logical continuation operation whose correctness-relevant execution targets cannot drift after `proposal_id` allocation.

---

# 1. Design handling of Round-2 findings

These are Design-thread incorporation statuses, not Review or Integration dispositions.

```yaml
XCONTRACT-R2-01:
  design_handling: INCORPORATED
  change: >
    Pre-activation rollover bootstrap now requires an enforceable adapter/runtime capability:
    either conversation reservation without execution, or execution-disabled bootstrap.
    Prompt wording alone is explicitly insufficient. Unsupported adapters may not use
    send-before-CAS activation.

XCONTRACT-R2-02:
  design_handling: INCORPORATED
  change: >
    ContinuationProposal now freezes action intent plus every correctness-relevant target.
    Retry under the same proposal_id may not re-resolve current/latest targets.

XCONTRACT-R2-03:
  design_handling: INCORPORATED
  change: >
    Rollover safe boundary is defined in execution-quiescence terms.
    Durable semantic State Proposals are not required to be reduced or discarded before rollover.

XCONTRACT-R2-04:
  design_handling: INCORPORATED
  change: >
    Reviewer-specific Context Projection / prompt / execution-input provenance is owned by
    Review Execution Provenance artifacts, not ReviewSnapshot target identity.

XCONTRACT-R2-05:
  design_handling: INCORPORATED
  change: >
    REFRESH freezes exact Browser Session / Adapter Attachment target identity and performs
    runtime stale validation before execution. It may not refresh a newly resolved current target
    under the old proposal_id.

XCONTRACT-R2-06:
  design_handling: INCORPORATED
  change: >
    Carrier activation uses one generic compare-and-set contract with an explicit nullable
    expected-current carrier. Initial bind is null -> C1; rollover is C1 -> C2.
```

---

# 2. ContinuationProposal is one immutable logical runtime operation

A ContinuationProposal means:

> Perform this exact continuation action, for this exact Logical Thread, against these exact correctness-relevant targets and preconditions.

Minimum envelope:

```yaml
continuation_proposal:
  proposal_id: required
  run_id: required
  logical_thread_id: required

  action:
    CONTINUE_FOCUSED | COMPACT | REFRESH | ROLLOVER

  creation_basis:
    state_version: required

  expected_active_carrier_id:
    # action-specific required/nullable semantics defined below

  execution_target:
    # action-specific immutable tagged union

  operation_fingerprint: required
  created_at: required
```

Run State basis identity is therefore:

```text
(run_id, creation_basis.state_version)
```

consistent with the committed Run-owned State model.

## 2.1 Immutability invariant

Once `proposal_id` is allocated:

```text
action
run_id
logical_thread_id
creation_basis
expected_active_carrier_id
all correctness-relevant execution_target fields
operation_fingerprint
```

are immutable.

If any of those values must change, the result is a new logical operation and MUST receive a new `proposal_id`.

Forbidden retry behavior:

```text
proposal CP-17 originally targets C1 / checkpoint K8 / projection P4
retry CP-17
-> re-resolve current carrier C2
-> re-resolve latest checkpoint K9
-> compile fresh projection P5
-> execute anyway
```

That is not a retry. It is a different operation masquerading under the same idempotency key.

## 2.2 Operation fingerprint

`operation_fingerprint` covers the canonicalized correctness-relevant ContinuationProposal envelope.

Conceptually:

```text
hash(
  action
  + run/thread identity
  + correctness-relevant state basis
  + expected carrier identity
  + action-specific immutable target descriptors
)
```

The concrete hash algorithm/serialization is implementation-defined.

Required invariant:

```text
same proposal_id + different operation_fingerprint
-> invariant violation / ID reuse
```

The fingerprint is an integrity aid. `proposal_id` remains the logical operation / future Execution Journal correlation key.

## 2.3 Retry rule

A retry of the same logical operation:

```text
-> reuse same proposal_id
-> reload the same immutable envelope
-> validate current preconditions against the frozen target
-> never substitute latest/current target identities
```

If a frozen correctness target is no longer valid, the operation becomes stale or ineligible.

The retry path does not mutate the old proposal into a new one.

A later Continuation evaluation may produce a new proposal with a new id.

This contract intentionally does not define low-level `planned / sent / observed / reconciled` retry states. Those remain Execution Journal scope.

---

# 3. Action-specific immutable execution targets

The generic rule is:

> Any input whose change can change what external/runtime operation will be performed is part of ContinuationProposal identity and must be frozen before `proposal_id` is reused for retries.

The following fields are the v0 minimum.

---

## 3.1 CONTINUE_FOCUSED

Purpose:

> Send one focused continuation request to the currently authorized active carrier.

Required frozen targets:

```yaml
execution_target:
  kind: CONTINUE_FOCUSED

  expected_active_carrier_id: required non-null

  continuation_input_ref:
    required: true
    semantics: >
      Immutable/content-addressed execution-input artifact that identifies the exact
      correctness-relevant provider request content or all immutable material needed to
      reproduce that exact request.
```

`continuation_input_ref` may point to an exact compiled request artifact, or to an immutable request spec whose closure includes all correctness-relevant inputs such as:

```text
Context Projection revision/content identity
next_intent payload
instruction/prompt template version
provider request shape when semantically relevant
```

It MUST NOT mean:

```text
"compile against latest context when retry happens"
```

Execution preconditions include:

```text
current active carrier for logical_thread_id == expected_active_carrier_id
correctness-relevant state basis still satisfies policy for this operation
```

If a new state/projection/input is desired, create a new ContinuationProposal.

---

## 3.2 COMPACT

Purpose:

> Produce a compaction/context artifact from one exact source boundary without changing canonical State by authority shortcut.

Required frozen targets:

```yaml
execution_target:
  kind: COMPACT

  expected_active_carrier_id: required non-null

  source_checkpoint_id: required

  source_turn_range:
    provider_conversation_id: required
    start_turn_ref: required
    end_turn_ref: required

  compaction_input_ref: required
```

`start_turn_ref` / `end_turn_ref` are immutable observed Turn identities/fingerprints or an equivalent exact immutable range descriptor.

The compactor may output:

```text
Compaction Artifact
Context Projection
optional separate State Delta Proposal
```

but the ContinuationProposal does not grant the compactor canonical State mutation authority.

Retry under the same `proposal_id` reads the same exact source range/checkpoint. It may not expand to newly arrived turns.

If new turns should be included, allocate a new proposal.

---

## 3.3 REFRESH

Purpose:

> Repair/recreate the exact Browser Session / Adapter Attachment runtime surface without changing Provider Conversation carrier identity.

Required frozen targets:

```yaml
execution_target:
  kind: REFRESH

  expected_active_carrier_id: required non-null

  browser_target:
    browser_session_id: required
    adapter_attachment_id: required
    attached_provider_conversation_id: required
```

The three identities must describe one exact runtime target at proposal creation.

The proposal does not mean:

```text
refresh whichever browser tab/session is current at retry time
```

Runtime stale behavior is specified in §8.

---

## 3.4 ROLLOVER

Purpose:

> Replace one Logical Thread's active Provider Conversation carrier while preserving Run and Logical Thread identity.

Required frozen targets:

```yaml
execution_target:
  kind: ROLLOVER

  expected_predecessor_id: required non-null

  checkpoint_id: required

  predecessor_turn_boundary_ref: required

  context_projection_ref:
    required: true
    semantics: immutable/versioned exact projection revision

  bootstrap_input_ref:
    required: true
    semantics: immutable exact bootstrap/context-establishment input

  destination_carrier_spec_ref:
    required: true
    semantics: >
      Immutable execution spec for provider/adapter/model destination details that are
      correctness-relevant to carrier creation.

  activation_strategy:
    RESERVE_THEN_CAS | GUARDED_BOOTSTRAP_THEN_CAS
```

The ROLLOVER state basis is correctness-relevant and therefore execution requires exact:

```text
(run_id, creation_basis.state_version)
```

plus exact expected predecessor.

A ROLLOVER retry may not:

```text
use latest checkpoint
compile a fresh Context Projection
switch destination provider/adapter
change bootstrap strategy
replace expected predecessor with current carrier
```

under the same `proposal_id`.

Any such change creates a new logical operation.

---

# 4. Correctness-relevant target closure

For every action-specific ref above, an immutable ID is sufficient only if it resolves to immutable/recoverable content or an exact immutable runtime identity as appropriate.

Examples of valid execution-input references:

```text
immutable artifact revision ID
content-addressed artifact
artifact_id + immutable revision_id
immutable request-spec revision + content hash
exact TurnRef with provider message identity/fingerprint
exact BrowserSession/Attachment lifecycle identity
```

Forbidden correctness target references inside a finalized ContinuationProposal include:

```text
latest checkpoint
current projection
current browser session
active tab
latest prompt template
current carrier
latest turn
```

unless the value has already been resolved and frozen into an exact identity before proposal finalization.

The same principle as immutable review targets applies conceptually, but this document does not modify or reopen XCONTRACT-03.

---

# 5. Rollover safe boundary = execution quiescence, not semantic settlement

The previous candidate required:

```text
relevant State Delta already reduced or explicitly discarded
```

as a rollover safe-boundary condition.

This revision removes that requirement.

A durable semantic State Proposal can remain pending across carrier rollover.

Rollover is safe when the old carrier's execution boundary is known and quiescent enough to prevent ambiguous concurrent continuation.

## 5.1 Required execution-oriented boundary

Before a ROLLOVER operation may proceed to carrier creation/activation, all of the following must hold:

```text
1. target Logical Thread is still runnable for this operation.

2. expected predecessor is still the active carrier.

3. predecessor provider generation/streaming is not active.

4. there is no known unresolved external dispatch ambiguity that could still produce
   an additional authoritative worker result on the predecessor.

5. the predecessor turn boundary referenced by the proposal is durably observed/captured.

6. checkpoint_id exists and is tied to a known canonical State revision and known thread/turn boundary.

7. context_projection_ref is already compiled/frozen against the proposal's exact basis.

8. carrier-touching execution for this Logical Thread is single-flight.

9. immediately before activation CAS, runtime revalidates the frozen state/carrier preconditions.
```

If runtime cannot name a stable predecessor boundary because the old carrier may still be producing an unobserved result, rollover is not safe.

That is an execution ambiguity, not a reason to guess.

## 5.2 Pending semantic proposals survive as durable artifacts

Rollover MUST NOT require this cleanup rule:

```text
pending semantic proposal
-> reduce or discard merely to allow carrier switch
```

Instead:

```text
pending State Proposal remains durable and immutable
rollover may advance canonical State through carrier-binding CAS
later State Proposal processing follows normal State Delta stale/revalidation semantics
```

The carrier lifecycle does not own the semantic proposal lifecycle.

Rollover therefore does not silently:

```text
cancel
rewrite
rebase
approve
deny
```

pending semantic proposals.

This may conservatively make an older pending State Proposal stale after unrelated canonical State advancement. That is handled by existing State Delta/revalidation policy, not by forcing semantic settlement inside rollover.

---

# 6. Enforceable pre-activation bootstrap capability invariant

The critical rule is:

> A provisional new Provider Conversation has no Logical Thread execution authority before active-binding CAS succeeds.

But this rule is only safe if pre-CAS behavior is enforceable at the execution layer.

Prompt text such as:

```text
"do not use tools"
"do not change state"
```

is NOT a sufficient capability guarantee.

## 6.1 Allowed activation strategies

A ROLLOVER proposal freezes exactly one activation strategy.

### Strategy A — `RESERVE_THEN_CAS`

Adapter capability requirement:

```text
CONVERSATION_RESERVATION_WITHOUT_EXECUTION
```

Meaning:

```text
create/reserve Provider Conversation identity
without dispatching a model turn or authoritative worker execution
```

Sequence:

```text
validate proposal + safe boundary
-> reserve provisional carrier Cnew without execution
-> discover/register Cnew identity
-> atomic carrier-binding CAS(expected predecessor -> Cnew)
   -> fail: Cnew remains unbound residue; do not dispatch normal work
   -> success: Cnew becomes active
-> only now may normal/bootstrap execution occur on Cnew
```

This is the preferred strategy when supported.

### Strategy B — `GUARDED_BOOTSTRAP_THEN_CAS`

Used only when Provider Conversation identity cannot be obtained before a first request.

Adapter capability requirement:

```text
PREACTIVATION_EXECUTION_SIDE_EFFECTS_DISABLED
```

This must be an adapter/provider/runtime capability, not a model instruction convention.

The capability MUST guarantee, for the pre-activation bootstrap request:

```text
- no external write side effects;
- no tool/connector invocation with persistent external effects;
- no delegated worker/subagent action that can cause external effects;
- no canonical State mutation path;
- no normal worker-result ingestion from the bootstrap response;
- no State Proposal / Human Gate / completion claim may be generated from that quarantined bootstrap output;
- runtime can identify the created Provider Conversation and keep it provisional until CAS.
```

Pure model computation/text generation may occur if the adapter cannot create the carrier otherwise, but its output is classified as bootstrap/runtime residue rather than a normal authoritative worker Turn for semantic ingestion.

Sequence:

```text
validate proposal + safe boundary + capability
-> dispatch exact frozen bootstrap_input_ref under side-effect-disabled preactivation mode
-> discover/register provisional Cnew
-> quarantine bootstrap response from normal semantic ingestion
-> atomic carrier-binding CAS(expected predecessor -> Cnew)
   -> fail: Cnew remains unbound residue; no normal continuation
   -> success: Cnew becomes active
-> normal continuation may begin only after activation
```

## 6.2 Unsupported adapter fallback

If an adapter supports neither:

```text
CONVERSATION_RESERVATION_WITHOUT_EXECUTION
```

nor enforceable:

```text
PREACTIVATION_EXECUTION_SIDE_EFFECTS_DISABLED
```

then that adapter is NOT eligible for send-before-CAS rollover activation.

Required behavior:

```text
ROLLOVER execution does not send a speculative first prompt
-> return explicit ineligibility/failure such as ROLLOVER_BOOTSTRAP_UNSUPPORTED
-> leave active carrier unchanged
```

The orchestration layer may later choose a different provider/adapter, a manually controlled path, or another semantic action.

Continuation does not silently weaken the authority boundary.

## 6.3 Adapter capability may disappear

Adapter capability is revalidated at execution time.

If proposal strategy is `GUARDED_BOOTSTRAP_THEN_CAS` but the exact execution environment can no longer enforce the declared capability:

```text
-> do not switch strategy under same proposal_id
-> fail/invalidate execution attempt
-> new Continuation evaluation may create a new proposal
```

---

# 7. Active Provider Conversation binding uses generic CAS with explicit null predecessor

Logical Thread continues to own:

```text
0..1 active Provider Conversation
```

The carrier relation uses one compare-and-set semantic contract for both initial binding and rollover.

Conceptual reducer-managed operation candidate:

```yaml
op: compare_and_set_active_provider_conversation

run_id: required
logical_thread_id: required
base_state_version: required

expected_current_provider_conversation_id:
  required: true
  nullable: true

new_provider_conversation_id:
  required: true
  nullable: false
```

Atomic preconditions:

```text
current canonical state version == base_state_version
current active carrier exactly equals expected_current_provider_conversation_id
```

`null` is a real expected value, not "field omitted".

### Initial bind

```text
expected_current = null
new = C1
```

### Rollover

```text
expected_current = C1
new = C2
```

Atomic success:

```text
active binding becomes new carrier
new canonical State version is produced
predecessor/supersedes lineage recorded when predecessor is non-null
old carrier identity/provenance retained
```

Atomic failure:

```text
STALE_PRECONDITION / equivalent
active binding unchanged
no partial rebinding
```

A provisional carrier created before CAS does not become active merely because it exists.

If CAS fails, it remains unbound external residue and MUST NOT continue normal Logical Thread work.

This document proposes the generic CAS semantics; exact reducer operation naming remains an integration/promotion choice.

---

# 8. REFRESH exact runtime target and stale behavior

REFRESH does not change Provider Conversation carrier identity.

It targets one exact Browser Session / Adapter Attachment lifecycle identity.

Before executing the external refresh/rebuild action, runtime MUST validate:

```text
1. browser_target.browser_session_id is still the intended lifecycle target;
2. browser_target.adapter_attachment_id is still the attachment this proposal targets;
3. attached_provider_conversation_id still equals proposal.expected_active_carrier_id;
4. Logical Thread active carrier still equals proposal.expected_active_carrier_id;
5. no replacement attachment/session has already superseded the frozen target in a way
   that would make this proposal act on a different runtime surface.
```

If the exact target can no longer be validated:

```text
-> RUNTIME_TARGET_STALE (or equivalent explicit stale result)
-> do not re-resolve "current browser session"
-> do not refresh a replacement target under the old proposal_id
```

A new Continuation evaluation may capture the new exact runtime target and create a new REFRESH proposal.

This is intentionally conservative.

REFRESH stale validation is runtime-target validation; it is not a new canonical State disposition and does not require a new Runtime Object type.

---

# 9. ROLLOVER execution sequence

This section freezes the contract-level ordering without defining Execution Journal step schema.

Common beginning:

```text
1. load immutable ContinuationProposal
2. validate operation_fingerprint
3. validate exact Run / Logical Thread identity
4. validate execution-oriented safe boundary
5. validate current State basis required by ROLLOVER
6. validate active carrier == expected_predecessor_id
7. validate frozen checkpoint/projection/bootstrap/destination refs
8. validate exact adapter capability for frozen activation_strategy
```

Then follow one of §6's two strategies.

The activation authority point is exactly:

```text
successful reducer-managed active-carrier CAS
```

Before that point:

```text
new carrier = provisional / unbound
```

After that point:

```text
new carrier = active carrier for Logical Thread
```

No bootstrap content, browser URL, creation timestamp, or "most recently created conversation" can substitute for this activation relation.

---

# 10. Stale and ineligibility semantics for Continuation operations

This contract defines only contract-level validation outcomes, not the future Execution Journal lifecycle.

Representative outcomes:

```text
STALE_STATE_BASIS
STALE_ACTIVE_CARRIER
RUNTIME_TARGET_STALE
UNSAFE_ROLLOVER_BOUNDARY
ROLLOVER_BOOTSTRAP_UNSUPPORTED
ADAPTER_CAPABILITY_UNAVAILABLE
INVALID_OPERATION_FINGERPRINT
```

Core rule:

> Stale or ineligible execution never mutates the immutable ContinuationProposal into a new target.

Forbidden:

```text
STALE_ACTIVE_CARRIER
-> replace expected carrier with current carrier
-> retry with same proposal_id
```

Forbidden:

```text
RUNTIME_TARGET_STALE
-> find newest Browser Session
-> refresh it with same proposal_id
```

Forbidden:

```text
STALE_STATE_BASIS
-> update base_state_version in place
```

Instead:

```text
old operation remains immutable provenance
new decision/evaluation -> new ContinuationProposal if work is still needed
```

Stale operation status is a factual execution relation, not a semantic Review Issue disposition.

---

# 11. Reviewer-input provenance ownership

Committed XCONTRACT-03 defines Review Target identity separately from Reviewer Execution Provenance.

This candidate does not modify that baseline.

The Multi-Conversation Role Model should therefore assign reviewer-specific execution input provenance to an artifact-layer execution record associated with the Review Thread / Review result.

Conceptually:

```yaml
review_execution_provenance:
  review_execution_id: required
  review_snapshot_id: required
  reviewer_logical_thread_id: required

  review_dimension: required
  review_contract_ref: required

  context_projection_ref:
    required: true
    semantics: immutable/versioned exact Reviewer input projection

  reviewer_input_ref:
    required: true
    semantics: immutable exact prompt/request/input spec or equivalent reproducible closure

  provider_model_observation:
    optional

  created_at: required
```

Ownership rule:

```text
ReviewSnapshot
-> identifies what was reviewed

Review Execution Provenance
-> records what reviewer-specific input/execution context was used
```

Reviewer-specific Context Projection does NOT become Candidate target identity merely because it influenced reviewer reasoning.

This provenance artifact:

```text
- is not a new canonical Runtime Object class;
- has no Reducer bypass authority;
- does not mutate ReviewSnapshot;
- may be referenced by Review Issues / Review Result for reproducibility.
```

The exact storage schema may live with existing Review artifacts or generic execution provenance. This design only freezes ownership and semantic separation.

---

# 12. Multi-Conversation Role boundary remains unchanged

Control / Design / Review / Integration remain reasoning/orchestration roles.

They do not gain carrier-binding or canonical mutation authority from role name.

ContinuationProposal execution is still runtime-controlled per Logical Thread.

Role output and runtime continuity remain separate:

```text
Design role says:
  "Candidate revision ready for review"

Continuation layer says:
  "this Design Thread's current carrier needs REFRESH"
```

or:

```text
Review role produces Review Issues

Continuation layer may ROLLOVER that Review Thread's carrier
without changing the ReviewSnapshot target
```

Rollover changes carrier, not logical role identity, Work Item identity, Review target identity, or canonical semantic authority.

---

# 13. Counterexamples and expected behavior

## 13.1 Retry cannot retarget latest projection

Initial operation:

```text
CP-10
ROLLOVER
state v31
expected predecessor C1
checkpoint K8
projection P4
```

Before retry:

```text
state becomes v32
new checkpoint K9 exists
projection P5 exists
```

Invalid retry:

```text
reuse CP-10 with v32/K9/P5
```

Correct behavior:

```text
CP-10 fails stale validation if its preconditions no longer hold
new evaluation may create CP-11 targeting v32/K9/P5
```

## 13.2 Two concurrent rollovers

Initial:

```text
state v40
active carrier C1
```

Two immutable operations:

```text
A expects C1 -> C2
B expects C1 -> C3
```

Only one active-binding CAS can succeed.

The loser cannot retarget its expected predecessor to the winner under the old proposal id.

## 13.3 Provider requires first send for Conversation ID but cannot disable side effects

Adapter capabilities:

```text
reserve without execution = false
preactivation side effects disabled = false
```

Correct behavior:

```text
ROLLOVER_BOOTSTRAP_UNSUPPORTED
active carrier remains predecessor
no speculative first prompt is sent
```

A prompt saying "please do nothing except acknowledge" does not satisfy the contract.

## 13.4 REFRESH target already replaced

Proposal:

```text
browser session B1
attachment A1
carrier C7
```

Before execution runtime now has:

```text
browser session B2
attachment A2
carrier C7
```

Correct behavior:

```text
old REFRESH proposal -> RUNTIME_TARGET_STALE
```

not:

```text
refresh B2/A2 under old proposal_id
```

## 13.5 Pending semantic proposal during rollover

Before rollover:

```text
State Proposal SP-91 exists, pending authorization
carrier C1
```

Rollover safe boundary is otherwise satisfied.

Correct behavior:

```text
SP-91 remains durable
ROLLOVER does not reduce/discard it merely to become safe
carrier CAS may proceed if its own exact preconditions hold
SP-91 later follows normal State Delta stale/revalidation semantics
```

## 13.6 Initial carrier binding

Logical Thread has no active carrier:

```text
active = null
```

Use the same CAS semantic:

```text
expected_current = null
new = C1
```

No separate "initial bind" mutation contract is required.

---

# 14. Authority boundary

This revision adds no new canonical authority.

Specifically:

```text
ContinuationProposal
!= State Proposal
!= AuthorizationResult
!= Authorized Delta
```

A ContinuationProposal may cause runtime/provider actions when the execution layer is allowed to act, but it cannot directly:

```text
commit a decision
change Run scope
close an Open Question in canonical State
approve a Human Gate
rewrite a pending State Proposal
mark Run complete
```

Carrier-binding CAS remains reducer-managed because the active carrier relation is canonical NOOS-owned runtime state.

Compaction remains projection/artifact authority.

Review roles remain advisory/reasoning roles.

Bootstrap capability only constrains execution; it does not create semantic authority.

---

# 15. No new durable semantic state

This candidate does not reintroduce an authoritative Continuation state machine containing independent truths such as:

```text
RUNNING
WAITING_HUMAN
COMPLETE
ROLLOVER_PENDING
```

If a concept has semantic workflow meaning, canonical Run State / semantic proposal governs it.

If a concept is transient external-execution progress, it belongs to runtime observation and eventually Execution Journal.

The immutable ContinuationProposal is an operation artifact, not a second State Store.

---

# 16. Explicit non-goals

This candidate deliberately does NOT define:

```text
Execution Journal event names
planned/sent/observed/reconciled state machine
provider request receipt schema
crash restart algorithm
residue garbage collection policy
exact browser automation APIs
adapter-specific capability detection implementation
full compaction artifact schema
new semantic State Proposal rebase algorithm
```

Those remain downstream work after this contract reaches promotion maturity.

---

# 17. Assumptions

- Runtime Object Model & Authority Model v0.1 remains Baseline.
- State Delta + Reducer Contract v0.1 remains Baseline.
- Run / Logical Thread are durable NOOS-owned identities.
- one Logical Thread has 0..1 active Provider Conversation.
- active carrier relation changes are reducer-managed CAS operations.
- Provider Conversation can exist while not active.
- Browser Session / Adapter Attachment lifecycle identities can be observed precisely enough to detect replacement/staleness.
- immutable/versioned Context Projection and execution-input artifacts can be persisted.
- adapter capability metadata can distinguish reservation-without-execution, side-effect-disabled preactivation execution, and unsupported behavior.
- Execution Journal can later use `proposal_id` as operation correlation/idempotency key without requiring this candidate to define journal steps now.

---

# 18. Remaining implementation choices

None of these choices should change the contract invariants above:

- exact `operation_fingerprint` serialization/hash scheme;
- exact artifact schema for `continuation_input_ref` / `bootstrap_input_ref`;
- exact names for adapter capability enums;
- exact reducer operation name for active-carrier CAS;
- exact stale/ineligibility error-code spelling;
- whether Review Execution Provenance is stored as a standalone artifact or embedded in a generic Review Execution record;
- provider-specific mechanism used to enforce side-effect-disabled preactivation execution.

---

# 19. Proposed contract changes

Relative to the previous cross-contract candidate, this revision proposes only the following narrow tightenings:

1. ContinuationProposal becomes an immutable execution envelope after `proposal_id` allocation.
2. Retry under one `proposal_id` never re-resolves correctness-relevant `latest/current` targets.
3. CONTINUE_FOCUSED freezes exact continuation input and expected carrier.
4. COMPACT freezes exact source checkpoint/range and compaction input.
5. REFRESH freezes exact Browser Session / Adapter Attachment / expected carrier and rejects stale runtime targets.
6. ROLLOVER freezes exact predecessor, State basis, checkpoint, turn boundary, Context Projection, bootstrap input, destination spec, and activation strategy.
7. Rollover safe boundary depends on execution quiescence, not mandatory semantic State Proposal settlement.
8. Pre-CAS bootstrap requires reservation-without-execution or enforceable side-effect-disabled capability; prompt-only non-authority is rejected.
9. Unsupported adapters may not use speculative send-before-CAS activation.
10. Carrier binding uses one generic CAS with explicit nullable expected-current carrier.
11. Reviewer-specific Context Projection/input provenance belongs to Review Execution Provenance, not ReviewSnapshot target identity.
12. No Runtime Object / Authority / XCONTRACT-03 semantics are reopened.

---

# 20. Targeted review questions

Independent Reviewer should focus on the following narrow questions:

1. Does the immutable ContinuationProposal envelope fully close the `proposal_id` + retry-retargeting gap for all four actions?
2. Are the action-specific required targets sufficient, or is any correctness-relevant target still allowed to remain as mutable `latest/current`?
3. Is the preactivation capability invariant strong enough to make send-before-CAS bootstrap genuinely non-authoritative at the execution layer rather than merely by prompt convention?
4. Is the unsupported-adapter fallback safe and sufficiently explicit?
5. Does the execution-quiescence safe boundary allow durable pending semantic proposals to survive rollover without creating split-brain provider execution?
6. Are REFRESH target identity and runtime stale semantics exact enough to prevent stale proposal retargeting?
7. Does nullable expected-current CAS correctly cover both initial binding and rollover without introducing a second binding version token?
8. Is reviewer-specific execution provenance correctly separated from committed XCONTRACT-03 Review Target identity?
9. Does any part of this revision accidentally design Execution Journal or create new canonical mutation authority?
10. Is any remaining issue promotion-blocking for Issue #3?

Requested disposition: targeted re-review only. Do not infer approval from Design incorporation of R2-01 through R2-06.
