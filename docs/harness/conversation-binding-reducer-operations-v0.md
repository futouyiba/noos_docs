# NOOS Harness — Conversation Binding Reducer Operations v0

> Status: Working Design Contract / V1 operational-state extension
>
> Purpose: make Current Conversation Binding, actuation-lease transfer, and submission-dispatch claiming concrete Reducer operations that compete inside one atomic authority/state boundary.
>
> This is a narrow extension of `state-delta-reducer-contract.md`. It does not introduce a new authority system. All operations below are ordinary Proposal → Policy → Authorized Delta → Reducer transitions; V1 policy may machine-authorize them when deterministic preconditions pass.

## 1. Why this extension exists

A conceptual rule such as:

```text
check current binding
check lease holder
click Send
```

is not sufficient. The binding or lease can change after the checks and before browser actuation.

V1 therefore freezes one stronger rule:

> Starting an automated provider dispatch, switching the current conversation binding, and transferring the actuation lease must compete through the same atomic Reducer state boundary.

Whichever transition commits first changes the preconditions observed by the others.

## 2. Canonical operational records

### 2.1 CurrentConversationBinding

```text
CurrentConversationBinding
- logical_thread_id
- provider_conversation_ref
- binding_generation
```

It is the only authoritative current Logical Thread → Provider Conversation relation.

The committed current-binding relation must enforce both uniqueness directions inside the same Reducer transaction:

```text
logical_thread_id -> at most one current provider_conversation_ref
provider_conversation_ref -> at most one current logical_thread_id
```

Reverse lookups and ACTIVE/SUPERSEDED presentation states are projections, not separately writable truth.

### 2.2 ActuationLeaseAuthority

The lease used for automated dispatch authorization is also authoritative operational state, not a content-script-local fact.

```text
ActuationLeaseAuthority
- provider_conversation_ref
- binding_generation
- carrier_ref
- lease_generation
- status: ACTIVE | RELEASED
```

At most one ACTIVE lease exists for a `(provider_conversation_ref, binding_generation)` pair.

Browser/runtime attachment observations may be ephemeral, but an automated sender is authorized only by the canonical lease record.

### 2.3 SubmissionOperation dispatch fence

The existing SubmissionOperation lifecycle is extended with an immutable dispatch fence once dispatch is claimed:

```text
DispatchFence
- provider_conversation_ref
- binding_generation
- carrier_ref
- lease_generation
```

The fence records the exact binding/lease authority under which this one operation was allowed to begin browser actuation.

## 3. Operation: `commit_current_conversation_binding`

Conceptual operation envelope:

```yaml
op: commit_current_conversation_binding

target:
  logical_thread_id: L1

payload:
  expected_current_conversation_ref: C1   # null for initial bind
  expected_binding_generation: 7
  target_conversation_ref: C2
  target_preactivation_receipt_ref: PE-42
  reason: INITIAL | ROLLOVER | ADOPT | RECOVERY_REPAIR
  source_operation_id: OP-ROLLOVER-9
```

Reducer preconditions/invariants are evaluated atomically against the same tentative state:

1. canonical binding for `L1` equals `C1@g7` (or null@g0 for initial bind);
2. `C2` has a stable provider identity;
3. `C2` is not the current conversation of another Logical Thread;
4. `target_preactivation_receipt_ref` is valid for `C2` under the preactivation contract;
5. no nonterminal execution-owning SubmissionOperation is bound to the expected old binding;
6. the proposal/base state is not stale under the normal State Delta contract.

For condition 5, V1 treats these states as execution-owning/nonterminal:

```text
DISPATCHING
OBSERVED_ACCEPTED
UNCERTAIN
```

A PREPARED operation has not yet obtained dispatch authority. After a binding switch it may remain durable but any later `claim_submission_dispatch` against the old binding must fail and it may be cancelled/replanned.

On success the Reducer performs one transaction:

```text
CurrentConversationBinding(L1) := C2 @ g8
```

and persists the ordinary ApplyResult/Transition Record in the same local transaction.

Any existing lease for the old binding becomes ineligible by binding-generation mismatch. It need not be independently trusted as revoked truth.

No partial forward/reverse mutation is allowed.

## 4. Operation: `transfer_actuation_lease`

Conceptual operation:

```yaml
op: transfer_actuation_lease

target:
  provider_conversation_ref: C2

payload:
  binding_generation: 8
  expected_carrier_ref: tab-207       # null when assigning first lease
  expected_lease_generation: 3
  target_carrier_ref: tab-233
```

Atomic preconditions:

1. canonical CurrentConversationBinding still points to `C2@g8`;
2. current ACTIVE lease matches the expected carrier/lease generation (or is absent for first assignment);
3. target carrier is currently attached to C2 and provider identity is resolved;
4. there is no execution-owning SubmissionOperation whose DispatchFence targets `C2@g8`;
5. normal Proposal/base-state concurrency checks pass.

On success:

```text
lease_generation := expected_lease_generation + 1
ACTIVE lease := target_carrier_ref
```

The previous carrier loses automated actuation authority immediately from the Reducer's perspective.

## 5. Operation: `claim_submission_dispatch`

This operation closes the check-before-use race.

A browser adapter MUST NOT begin automated submit/click actuation merely because it locally observed READY and a matching lease.

It first obtains a durable dispatch claim.

Conceptual operation:

```yaml
op: claim_submission_dispatch

target:
  submission_operation_id: SUB-81

payload:
  logical_thread_id: L1
  provider_conversation_ref: C2
  expected_binding_generation: 8
  carrier_ref: tab-233
  expected_lease_generation: 4
```

Atomic preconditions:

1. SubmissionOperation `SUB-81` is `PREPARED`;
2. its target Logical Thread / Provider Conversation agree with the payload;
3. canonical CurrentConversationBinding is exactly `L1 -> C2@g8`;
4. canonical ACTIVE ActuationLeaseAuthority is exactly `C2@g8 -> tab-233@lease4`;
5. no other SubmissionOperation for `C2@g8` owns unresolved automated execution (`DISPATCHING`, `OBSERVED_ACCEPTED`, or `UNCERTAIN`);
6. carrier runtime evidence required by the operation policy is eligible (normally READY, with provider identity stable);
7. normal State Delta base-version/precondition checks pass.

On success, in the same Reducer transaction:

```text
SUB-81.state := DISPATCHING
SUB-81.dispatch_fence := (C2, g8, tab-233, lease4)
```

The persisted ApplyResult is the dispatch permit for this operation.

Only after receiving that successful ApplyResult may the Shuttle/browser adapter perform the one blind provider actuation allowed by the Submission Idempotency Contract.

## 6. Atomic competition: dispatch vs rollover

This is the key race closure.

Suppose L1 currently binds to `C1@g7`.

### Dispatch claim commits first

```text
claim SUB-90
→ SUB-90 = DISPATCHING on C1@g7
```

A concurrent rollover proposal now sees an execution-owning SubmissionOperation on the old binding and `commit_current_conversation_binding` rejects its precondition.

The rollover may be retried only after SUB-90 reaches a non-execution-owning terminal/reconciled state.

### Rollover binding commit commits first

```text
L1: C1@g7 -> C2@g8
```

A concurrent/stale dispatch claim targeting C1@g7 now fails its canonical-binding precondition.

Therefore there is no state in which both operations can obtain valid authority from the Reducer.

## 7. Atomic competition: dispatch vs lease transfer

The same rule applies to duplicate-tab carrier transfer.

### Dispatch claim commits first

Lease transfer is rejected while the operation owns execution under the current lease.

### Lease transfer commits first

A stale dispatch claim containing the old `carrier_ref/lease_generation` is rejected.

This closes the stale-carrier check/use race without requiring browser tabs to coordinate directly.

## 8. When may binding/lease movement resume?

Binding switch and lease transfer remain blocked while a SubmissionOperation is in:

```text
DISPATCHING
OBSERVED_ACCEPTED
UNCERTAIN
```

This intentionally includes `OBSERVED_ACCEPTED`: provider acceptance does not mean Agent execution is complete. Rollover must not create a new executable carrier while the old conversation may still be generating or invoking tools.

Once the operation is reconciled to a non-execution-owning state such as:

```text
COMPLETED
FAILED_SAFE
CANCELLED
```

normal binding/lease transitions may proceed.

If the operation remains `UNCERTAIN`, V1 fails closed rather than moving the binding around an unresolved provider-side effect.

## 9. Browser actuation rule

A browser sender may actuate only when all of the following are true:

```text
successful ApplyResult for claim_submission_dispatch
AND
SubmissionOperation is still DISPATCHING under the same DispatchFence
AND
adapter is the carrier named in that DispatchFence
```

A local `READY` observation, matching URL, matching title, or stale lease cache is never sufficient authority.

The provider click/submit is therefore downstream of a durable Reducer grant, not upstream of it.

## 10. Human manual interaction

This contract fences Harness automation, not the user's physical keyboard/mouse.

If Human manual submission occurs while an automated operation is PREPARED/DISPATCHING/UNCERTAIN, the provider evidence may become ambiguous. The Submission Idempotency Contract already requires conservative reconciliation/pause.

V1 does not claim transactional exclusion over Human UI actions.

## 11. Preactivation receipt dependency

`commit_current_conversation_binding` requires a valid provider-specific preactivation eligibility receipt.

The exact conformance criteria are defined by the Current Conversation Binding Contract. The Reducer does not infer from prompt text that a conversation was safely quarantined; it validates a structured receipt produced only by a conforming provider adapter/policy path.

## 12. Reducer implementation requirement

These are not advisory helper checks.

An implementation claiming safe V1 automated binding/rollover MUST implement the equivalent of:

```text
commit_current_conversation_binding
transfer_actuation_lease
claim_submission_dispatch
```

inside the same atomic state/Reducer transaction domain (or a formally equivalent serialized authority service with the same invariants and crash-consistent ApplyResult semantics).

Direct Shuttle mutation of binding/lease/dispatch authority is non-conforming.

## 13. Non-goals

This extension does not define:

- semantic Review acceptance;
- ResultDelivery idempotency;
- general distributed locking;
- multi-Designer authority;
- provider DOM selectors;
- Human exclusion from the browser UI.

## 14. Working conclusion

V1 closes stale-carrier and rollover/dispatch races by moving the right to *begin dispatch* into the same atomic Reducer boundary as binding and lease changes.

The safety primitive is not "check immediately before clicking". It is:

> atomically claim execution under a specific binding generation and lease generation; while that claim remains execution-owning, neither rollover nor lease transfer may commit.
