# Review Candidate: Continuation Operation Envelope & Rollover Bootstrap Tightening v4

- status: review_candidate
- date: 2026-08-29
- source_role: Designer-2 / Design Thread
- work_item: Issue #3 — Continuation operation envelope & rollover bootstrap tightening
- supersedes_review_target: `94ce5ded4306d67137b02b1de90af4f6a8cc99dd` for Issue #3 only
- base_commit: `d8ad8038a89c229421c94fa14d3a0cabad0d105b`
- base_candidate: `review-candidates/2026-08-28-continuation-operation-envelope-rollover-bootstrap-v3.md` @ `94ce5ded4306d67137b02b1de90af4f6a8cc99dd`
- committed_dependency: XCONTRACT-03 exact v5 semantics @ `39c7a9b755567792e343bec94a7dd8725c5d37a5`
- scope: provisional destination-carrier activation integrity plus fingerprint-profile recovery hygiene only
- explicitly_out_of_scope: reopening prior closed Issue #3 contracts, XCONTRACT-03, Runtime Object / Authority redesign, broad State Delta + Reducer redesign, Execution Journal schema, full crash recovery algorithm, provider-specific UI automation implementation

---

## 0. Normative composition

v4 is a **narrow additive revision** over the exact immutable v3 Candidate above.

Normative Issue #3 Candidate semantics for v4 are:

```text
exact v3 @ 94ce5ded4306d67137b02b1de90af4f6a8cc99dd
+
this v4 delta
```

All v3 rules remain unchanged unless this v4 file explicitly tightens them.

The exact v3 commit is therefore part of v4's immutable target-semantic closure and MUST NOT be replaced by branch head, latest Candidate, or a later v3-like file.

This format intentionally avoids rewriting already-reviewed v3 text and reduces the chance of regression while preserving one exact reviewable semantic closure.

---

# 1. Design handling of Review #10 findings

These are Design-thread incorporation statuses only. They are not Reviewer or Integration dispositions.

```yaml
XCONTRACT-R5-01:
  design_handling: INCORPORATED
  change: >
    ROLLOVER now freezes and revalidates an exact provisional destination-carrier boundary.
    Same Cnew identity is no longer sufficient for first-apply eligibility. After reservation or
    guarded bootstrap, runtime records the exact expected Cnew conversation/context boundary;
    every possible first apply must prove that Cnew has not accumulated any additional
    pre-activation user/assistant/tool/subagent activity beyond that boundary. If the adapter
    cannot prevent such activity or detect it before first apply, autonomous rollover fails closed.

XCONTRACT-R5-02:
  design_handling: INCORPORATED
  change: >
    Canonicalization-profile migration may not silently turn recovery of an unresolved effective
    continuation operation into a fresh decision. Recovery first resolves existing effective
    Basis/Proposal identity under the persisted profile/mapping, or uses an explicit migration/
    orchestration epoch policy. Profile change alone is not permission to duplicate an unresolved
    logical operation.
```

No other v3 contract is reopened.

---

# 2. Core symmetry invariant

v3 already protects the predecessor from activation-window drift:

```text
same predecessor carrier ID
!= proof of same predecessor execution state
```

v4 applies the same principle to the provisional destination:

```text
same provisional carrier ID
!= proof of same destination conversation/context state
```

A ROLLOVER operation prepares one exact destination carrier **state boundary**, not merely one Provider Conversation identifier.

Therefore first-apply eligibility requires both sides of the carrier transition to remain valid:

```text
predecessor C1
  -> still exactly quiescent at frozen predecessor boundary

provisional Cnew
  -> still exactly at frozen pre-activation destination boundary
```

Only then may the exact Authorized carrier-CAS Delta be first-applied.

---

# 3. Provisional destination carrier boundary

## 3.1 Required runtime observation

After the exact provisional carrier has been created by the frozen activation strategy, but before the carrier-binding State Proposal may proceed toward first apply, runtime MUST establish an exact observed **provisional destination boundary**.

Conceptually:

```yaml
provisional_destination_boundary:
  continuation_proposal_id: required
  provisional_carrier_id: required

  boundary_kind:
    RESERVED_EMPTY | GUARDED_BOOTSTRAP_COMPLETE

  provider_conversation_boundary_ref: required immutable/recoverable

  bootstrap_exchange_ref:
    required_if: boundary_kind == GUARDED_BOOTSTRAP_COMPLETE

  observed_at:
    provenance_only
```

The exact physical schema/name is not normative.

What is normative is that the boundary gives runtime enough immutable/recoverable evidence to answer:

> Has this exact provisional Provider Conversation changed since the point at which this ROLLOVER operation finished preparing it for activation?

The boundary is runtime execution provenance/eligibility evidence. It is not canonical Run State and does not gain mutation authority.

## 3.2 `RESERVE_THEN_CAS`

For:

```text
activation_strategy = RESERVE_THEN_CAS
```

conversation identity is reserved without model/worker execution.

The expected destination boundary is the exact reserved/empty conversation lifecycle state observable after reservation.

Conceptually:

```text
Cnew reserved
-> no user turn
-> no assistant turn
-> no tool/subagent turn
-> freeze boundary Knew-0
```

The adapter need not use the literal name `Knew-0`; it must expose an equivalent exact lifecycle/history boundary.

## 3.3 `GUARDED_BOOTSTRAP_THEN_CAS`

For:

```text
activation_strategy = GUARDED_BOOTSTRAP_THEN_CAS
```

the side-effect-disabled quarantined bootstrap may create an initial exchange before carrier identity is known.

After that exact frozen bootstrap finishes, runtime MUST capture the exact resulting bootstrap boundary.

Conceptually:

```text
bootstrap_input_ref = BI-7
-> exact quarantined provider exchange on Cnew
-> observe bootstrap exchange boundary Knew-B1
-> freeze Knew-B1 as expected provisional destination boundary
```

The boundary MUST correspond to the exact bootstrap produced by this immutable ContinuationProposal and destination spec.

It MUST NOT mean:

```text
"whatever history Cnew has when activation eventually happens"
```

---

# 4. Destination-side prevent-or-detect guarantee

Between provisional destination boundary capture and canonical first apply, autonomous ROLLOVER requires an enforceable destination-side guarantee.

For all NOOS-controlled paths, runtime MUST prevent additional normal dispatch to provisional Cnew before activation.

For manual/external/provider activity capable of changing Cnew conversation/context, the exact adapter/runtime surface MUST guarantee one of:

```text
A. such pre-activation activity is prevented/disabled;

or

B. such activity is observed in time to invalidate activation eligibility before any first-apply carrier CAS may enter Reducer.
```

This includes, where the provider surface can produce them:

```text
new user messages
new assistant/model turns
extra bootstrap-like turns
persistent or nonpersistent tool turns that alter conversation context
subagent/delegated-worker turns represented in provider context
provider-side system/context mutation that changes the effective future conversation state
```

The rule is about **future-context correctness**, not merely persistent side effects.

A pre-activation message can violate destination integrity even if it makes no canonical State mutation and no external write, because later normal continuation would consume a changed provider conversation context.

If the runtime cannot prevent or reliably detect such destination activity before first apply:

```text
PROVISIONAL_CARRIER_INTEGRITY_UNSUPPORTED
or UNSAFE_ROLLOVER_BOUNDARY
-> autonomous rollover is not eligible
-> do not first-apply carrier CAS
```

Prompt wording is not a substitute.

---

# 5. Carrier Activation Apply Gate — v4 tightening

v3 §12 remains normative.

v4 adds destination-integrity predicates to every possible **first-apply** path.

Immediately before an as-yet-unapplied carrier-binding Authorized Delta may enter Reducer, the Carrier Activation Apply Gate MUST establish all v3 predicates **and**:

```text
13. the exact provisional destination boundary for this CP/Cnew is available and validates;

14. provisional Cnew still equals the exact new_provider_conversation_id named by the
    immutable State Proposal / Authorized Delta;

15. current observed Cnew conversation/history/context boundary exactly equals the frozen
    provisional destination boundary;

16. no new Cnew user turn has occurred after that boundary;

17. no new Cnew assistant/model/tool/subagent/context-changing provider activity has occurred
    after that boundary;

18. the destination-side prevent-or-detect capability remains enforceable through the
    first-apply critical section.
```

`14` alone is explicitly insufficient.

Required invariant:

```text
same Cnew ID
+ different observed destination boundary
-> PROVISIONAL_CARRIER_CHANGED / UNSAFE_ROLLOVER_BOUNDARY
-> do not first-apply
```

The unchanged Authorized Delta remains durable provenance but is not currently runtime-eligible for first apply.

Continuation does not rewrite the Delta to another carrier or another boundary.

---

# 6. Authorization wait / Human Gate

The destination boundary is subject to the same long-wait concern already applied to predecessor quiescence.

Counterexample:

```text
C1 quiescent
-> guarded bootstrap prepares C2 at boundary Knew-B1
-> State Proposal enters Human Gate
-> user manually opens C2 and sends message U2
-> Human approves
```

Even if:

```text
C2 ID is unchanged
C1 is still quiescent
State version still matches
Authorized Delta is exact
```

first apply MUST fail because:

```text
current C2 boundary != Knew-B1
```

Expected runtime result:

```text
PROVISIONAL_CARRIER_CHANGED
or UNSAFE_ROLLOVER_BOUNDARY
```

No carrier activation occurs.

If policy authorization waits for a long period, the gate always revalidates both:

```text
predecessor-side quiescence
+
provisional-destination integrity
```

after authorization and immediately before first Reducer submission.

---

# 7. Crash/restart semantics for provisional destination integrity

v3 already requires Authorized Delta with no successful ApplyResult to reacquire predecessor fence and rerun final quiescence before first apply.

v4 adds:

> Restart/recovery MUST also recover enough immutable execution provenance to identify the expected provisional destination boundary and compare it with the current observed Cnew boundary.

Conceptually:

```text
load exact CP
load exact State Proposal / Authorized Delta
check no successful ApplyResult
recover exact provisional Cnew identity
recover expected provisional destination boundary Knew
observe current Cnew boundary Know

Knew == KNow
  -> destination integrity may pass

Knew != KNow / cannot prove equality
  -> do not first-apply
```

Fail-closed rule:

```text
expected destination boundary unavailable or unrecoverable
-> runtime cannot prove first-apply eligibility
-> do not first-apply
```

This requirement does not define a complete Execution Journal schema.

Implementations may persist the boundary in an existing runtime execution-provenance record, operation artifact, adapter observation record, or later Journal structure, as long as the contract invariant is enforceable and recoverable.

---

# 8. Already-applied receipt replay remains unchanged

The destination-boundary checks govern only an operation that could cause **first apply**.

If Baseline State Store already contains the successful persisted transition/ApplyResult for the exact carrier-CAS Delta:

```text
same delta_id + same fingerprint + successful transition exists
-> return original persisted ApplyResult
-> Cnew is already canonical active carrier
-> do not require a new provisional-boundary gate to recreate a completed activation
```

This preserves v3 and Baseline idempotency.

Any post-activation change to Cnew is normal active-carrier runtime evolution, not a retroactive failure of the completed activation.

---

# 9. State Proposal / Authorized Delta remain unchanged

v4 does **not** add the provisional boundary to canonical carrier CAS identity or grant the boundary canonical mutation authority.

The canonical mutation remains exactly:

```text
State Proposal
-> Policy
-> Authorized Delta
-> Carrier Activation Apply Gate
-> Reducer
-> ApplyResult
```

with carrier CAS fields:

```text
logical_thread_id
expected_current_provider_conversation_id
new_provider_conversation_id
base_state_version in enclosing State Proposal/Delta
```

The provisional destination boundary is runtime first-apply eligibility evidence.

It constrains whether the already-authorized exact Delta may be submitted for first apply; it does not rewrite, extend, or reauthorize that Delta.

This preserves:

```text
LLM proposes; Policy authorizes; Reducer applies; NOOS records.
```

---

# 10. Normative ROLLOVER sequence — v4

v3 §20 is tightened to the following ordering:

```text
1. canonicalize semantic DecisionBasis payload
2. compute self-describing decision_basis_fingerprint
3. create/reuse DecisionBasis by (logical_thread_id, fingerprint)
4. create/reuse ContinuationProposal by same durable basis key
5. validate Basis→Proposal equality + operation_fingerprint + immutable target closure
6. validate ROLLOVER preparation predicates / State EXACT
7. acquire predecessor activation fence
8. reserve provisional carrier or guarded-bootstrap exact Cnew
9. observe + freeze exact provisional destination boundary Knew
10. enforce destination-side no-normal-dispatch / prevent-or-detect guarantee
11. create/reuse exact carrier-binding Baseline State Proposal linked to CP
12. Policy evaluates State Proposal
13. if not yet authorized, wait according to Baseline policy; no first apply occurs
14. once exact Authorized Delta exists, check whether successful ApplyResult already exists
    - yes -> use persisted receipt; do not recreate activation
    - no -> possible first-apply path continues
15. acquire/reconfirm current predecessor activation fence as needed
16. recover/reconfirm exact expected provisional destination boundary Knew
17. AFTER authorization/any wait, perform full predecessor quiescence revalidation
18. AFTER authorization/any wait, validate current Cnew boundary == Knew and no extra
    pre-activation destination activity occurred
19. pass Carrier Activation Apply Gate
20. submit unchanged exact Authorized Delta to Reducer while predecessor and destination
    critical-section guarantees remain enforceable
21. observe persisted ApplyResult
    - applied -> Cnew becomes canonical active carrier
    - rejected_stale/precondition/invariant -> Cnew remains unbound
22. release predecessor/destination activation restrictions after result/abort
23. normal continuation on Cnew only after exact ApplyResult.applied
```

No step may retarget Basis, CP, provisional boundary, State Proposal, or Authorized Delta to current/latest values.

---

# 11. Counterexamples and required outcomes

## 11.1 Reserved carrier receives a pre-activation message

```text
RESERVE_THEN_CAS
C2 reserved at empty boundary K0
Policy waits
manual user message U1 arrives on C2
```

Required:

```text
current C2 boundary != K0
-> PROVISIONAL_CARRIER_CHANGED
-> no first apply
```

Same `C2` ID does not save the operation.

## 11.2 Guarded bootstrap carrier receives another turn

```text
bootstrap exchange establishes C2 boundary KB1
before activation another assistant/user/tool turn appears
```

Required:

```text
current C2 boundary != KB1
-> UNSAFE_ROLLOVER_BOUNDARY
-> no first apply
```

The old CP/Authorized Delta remain immutable; execution does not silently accept the changed destination context.

## 11.3 Destination activity cannot be prevented or detected

Adapter can create Cnew, but provider UI permits external messages and NOOS cannot observe them before activation.

Required:

```text
PROVISIONAL_CARRIER_INTEGRITY_UNSUPPORTED
-> autonomous rollover ineligible
```

This is analogous to predecessor fence fail-closed behavior.

## 11.4 Crash after Cnew preparation

```text
C2 prepared at KB1
Authorized Delta durable
crash
no successful ApplyResult
```

Restart:

```text
recover expected KB1
observe current C2 boundary
-> equal: continue only after all v3 + v4 first-apply checks pass
-> different/unprovable: do not first-apply
```

## 11.5 Already-applied Delta

```text
carrier CAS applied
ApplyResult durable
caller lost receipt
```

Required:

```text
return persisted ApplyResult
```

No attempt is made to compare Cnew against the old provisional boundary because activation already completed.

---

# 12. Fingerprint-profile migration recovery hygiene

v3 §4.3 remains correct that canonicalization profile is part of fingerprint identity and cross-profile equality is not implicit.

v4 adds one narrow recovery invariant:

> A fingerprint-profile deployment change MUST NOT silently reinterpret recovery of an already-existing unresolved effective continuation operation as a fresh decision merely because recomputation under a new profile would yield a different fingerprint.

For recovery of unresolved work, a conforming implementation MUST do one of the following or an equivalent explicit policy:

```text
A. resolve/recover the existing effective DecisionBasis/ContinuationProposal under its persisted
   fingerprint profile before performing fresh evaluation;

B. preserve an explicit migration mapping from the old effective basis/proposal identity;

C. define profile migration as an explicit new orchestration epoch that deliberately terminates/
   supersedes prior unresolved effective operation identity under governance rules.
```

Forbidden implicit behavior:

```text
old unresolved:
profile-v1 -> F7 -> B42 -> CP17

restart after deployment:
profile-v2 -> F9
-> silently treat as unrelated basis
-> create CP18
```

Profile migration protocol/storage remains implementation/governance scope.

The only v4 invariant is that migration is explicit rather than an accidental crash-idempotency bypass.

---

# 13. Preserved v3 invariants

v4 does not modify the following reviewed v3 contracts:

```text
- DecisionBasis semantic canonical fingerprint domain;
- Basis create/reuse by (logical_thread_id, decision_basis_fingerprint);
- ContinuationProposal create/reuse on the durable basis key;
- same-basis/different-operation conflict;
- Basis→Proposal run/thread/state/carrier equality;
- one authoritative execution_target.expected_current_carrier_id;
- CONTINUE_FOCUSED = EXACT at final dispatch authorization;
- ROLLOVER = EXACT;
- COMPACT / REFRESH = PROVENANCE_ONLY;
- predecessor activation fence;
- post-authorization predecessor quiescence validation;
- Authorized Delta != permanent first-apply eligibility;
- successful ApplyResult replay vs first apply;
- explicit State Proposal -> Policy -> Authorized Delta -> Reducer authority bridge;
- Baseline rejected_stale / rejected_precondition / rejected_invariant vocabulary;
- guarded bootstrap side-effect-disabled + semantic-ingestion quarantine;
- unsupported bootstrap/fence fail-closed behavior;
- exact REFRESH target/no-retarget;
- pending semantic State Proposal survival across rollover;
- nullable null->C1 / C1->C2 carrier CAS;
- Review Execution Provenance separation from XCONTRACT-03 target identity;
- no cross-system State/send atomicity claim;
- no full Execution Journal state machine.
```

---

# 14. Authority and object boundary

No new canonical mutation authority is introduced.

```text
Provisional Destination Boundary
!= canonical Run State
!= State Proposal
!= AuthorizationResult
!= Authorized Delta
!= Reducer authority
```

It is immutable/recoverable runtime execution-provenance evidence used to prove first-apply eligibility.

The Carrier Activation Apply Gate remains a runtime routing/eligibility invariant, not Policy and not Reducer.

If the exact deployment cannot ensure that every carrier-CAS first-apply path passes both predecessor and destination integrity checks, autonomous rollover is nonconforming/fail-closed for that architecture unless a future stronger Reducer-verifiable activation mechanism is explicitly designed.

v4 does not introduce such a token.

---

# 15. Assumptions added by v4

In addition to v3 assumptions:

- provider/adapter runtime can observe an exact provisional destination conversation/context boundary after reserve/bootstrap, or autonomous rollover is fail-closed;
- the expected provisional boundary can be retained/recovered long enough to validate any later first-apply attempt;
- NOOS-controlled normal dispatch to provisional Cnew can be prevented before activation;
- external/manual destination activity can either be prevented or detected before first apply;
- the provider conversation/history observation is sufficiently precise to distinguish boundary-preserving identity from same-ID context drift.

---

# 16. Remaining implementation choices

v4 leaves open:

- exact schema/name/storage for provisional destination boundary evidence;
- exact provider-specific conversation-boundary representation;
- whether the boundary is a last observed TurnRef, history fingerprint, provider revision token, immutable exchange receipt, or equivalent exact identity;
- exact adapter mechanism for preventing/detecting external pre-activation destination activity;
- exact runtime error-code spelling;
- exact fingerprint-profile migration protocol.

These choices may vary only if they preserve the normative integrity/recovery invariants above.

---

# 17. Proposed contract changes relative to v3

v4 adds only:

1. Same provisional carrier ID is insufficient to prove same activation target state.
2. After reserve/bootstrap, runtime freezes an exact provisional destination carrier boundary.
3. `RESERVE_THEN_CAS` freezes the reserved/empty boundary.
4. `GUARDED_BOOTSTRAP_THEN_CAS` freezes the exact observed bootstrap-complete boundary.
5. Before any possible first apply, current Cnew history/context boundary must equal the frozen provisional boundary.
6. Any extra pre-activation user/assistant/tool/subagent/context-changing activity makes the operation runtime-ineligible.
7. Autonomous rollover requires destination activity to be preventable or detectable before first apply; otherwise fail closed.
8. Crash/restart with no successful ApplyResult must recover and revalidate the expected provisional destination boundary as well as predecessor quiescence.
9. Successful ApplyResult replay remains exempt because activation already completed.
10. Fingerprint-profile migration may not silently bypass unresolved Basis/Proposal recovery idempotency.
11. No canonical carrier-CAS field, authority path, State-basis mode, or prior closed contract is changed.

---

# 18. Targeted re-review questions

Independent Reviewer should attack only:

1. Does the provisional destination boundary close the same-Cnew-ID/different-context activation drift counterexample?
2. Are both activation strategies covered: reserved-empty boundary and guarded-bootstrap-complete boundary?
3. Can any user/assistant/tool/subagent/provider context change occur after boundary capture yet still pass the first-apply gate?
4. Is prevent-or-detect fail-closed behavior strong enough without requiring a new canonical object or full Journal schema?
5. After crash/restart with Authorized Delta but no ApplyResult, is expected destination-boundary recovery/revalidation mandatory and sufficient at contract level?
6. Does successful ApplyResult replay remain cleanly distinct from first-apply eligibility?
7. Does provisional-boundary evidence remain runtime provenance/eligibility rather than leaking into canonical mutation authority?
8. Does profile-migration hygiene avoid accidental duplicate unresolved operations without overdesigning migration protocol?
9. Did any v3-closed area regress?
10. Is any remaining issue promotion-blocking for Issue #3?

Requested disposition: targeted independent re-review only. Design incorporation does not imply closure.