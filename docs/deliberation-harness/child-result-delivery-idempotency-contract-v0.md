# NOOS Deliberation Harness — Child Result Delivery Idempotency Contract v0

> Status: Working Design Contract / V1 scope / Not a frozen Review Candidate
>
> Purpose: close the remaining V1 seam where a durable child result (especially an Independent Review Result) must be returned to the current Primary Design Logical Thread without duplicate insertion after restart, lost acknowledgement, or parent rollover.

## 1. Core decision

V1 does **not** introduce a second parallel delivery state machine.

Child-result delivery is one specialized `SubmissionOperation`:

```text
SubmissionOperation.kind = DELIVER_CHILD_RESULT
```

It reuses the existing submission safety primitives:

- persist-before-actuate;
- one blind dispatch;
- `claim_submission_dispatch`;
- DispatchFence;
- `UNCERTAIN` reconciliation;
- conservative no-blind-retry policy.

The only additional semantics are:

1. a stable logical delivery identity independent of browser binding generation;
2. create-or-get deduplication for that logical delivery;
3. a durable ResultDeliveryReceipt proving whether the result was inserted and whether the resulting parent turn completed.

## 2. Distinguish result identity, logical delivery identity, and dispatch target

These are three different things.

### 2.1 WorkerResult identity

The child produces an immutable durable result:

```text
WorkerResult
- result_id
- result_fingerprint
- child_thread_id
- parent_thread_id
- work_item_id
- result_ref or immutable return_payload_ref
- relevant revision / snapshot ref
```

For Review, `result_id` identifies the immutable Review Result bound to its frozen target.

Same `result_id` with a different result fingerprint is an invariant violation.

### 2.2 Logical delivery identity

The V1 identity of “return this result to this parent thread” is:

```text
ResultDeliveryKey
= (result_id, destination_logical_thread_id)
```

The delivery key MUST NOT include:

- provider conversation ref;
- Browser Carrier / tabId;
- binding generation;
- lease generation.

Reason:

> parent rollover changes the execution carrier, not the semantic identity of the return.

If binding generation were part of the dedup key, rollover from g7 to g8 would incorrectly permit a second logical delivery of the same Review Result.

### 2.3 Concrete dispatch target

The concrete Provider Conversation / binding generation / carrier / lease are selected only when the delivery operation attempts dispatch.

They belong to the DispatchFence, not to the durable logical delivery identity.

## 3. Exactly-once claim

ChatGPT Web does not provide provider-native transactional exactly-once insertion.

Therefore V1 does not claim strict exactly-once provider delivery.

The contract is:

> exactly one durable logical delivery intent + at most one blind provider dispatch at a time + observable reconciliation + no automated second insertion when acceptance is proven or ambiguous.

This is effectively-once / duplicate-resistant delivery, consistent with the Command Submission / Idempotency Contract.

## 4. Create-or-get delivery operation

When a child reaches `RESULT_READY` and its operation contract requires return into the parent conversation, NOOS creates or retrieves a delivery operation by `ResultDeliveryKey`.

Conceptually:

```text
SubmissionOperation
- operation_id
- operation_kind: DELIVER_CHILD_RESULT
- work_item_id
- source_child_thread_id
- destination_logical_thread_id
- result_id
- result_fingerprint
- result_delivery_key
- payload_ref / payload_fingerprint
- state
- dispatch_fence?       # absent until dispatch claim succeeds
- pre_submit_baseline?  # recorded for the concrete attempt
```

Durable uniqueness rule:

```text
same ResultDeliveryKey + same result_fingerprint
→ return the existing SubmissionOperation / receipt
→ do not create another logical delivery

same ResultDeliveryKey + different result_fingerprint
→ rejected_invariant
```

The caller does not need to remember the previous `operation_id` after crash in order to prevent duplicate logical delivery; the delivery key can recover it.

## 5. Destination is late-bound

A `DELIVER_CHILD_RESULT` operation targets a Logical Thread, not a fixed Provider Conversation.

Before dispatch claim:

```text
resolve destination_logical_thread_id
→ read canonical CurrentConversationBinding
→ read canonical ActuationLeaseAuthority
→ verify Carrier READY / provider identity stable
→ attempt claim_submission_dispatch
```

The operation remains the same logical delivery even if the parent rolled over before dispatch.

### Example

```text
Review Result R17
→ delivery key = (R17, Design L1)

operation PREPARED while L1 -> C1@g7

before dispatch claim:
L1 rolls over -> C3@g8

same delivery operation
→ resolves current L1 binding again
→ claims dispatch against C3@g8
```

No new delivery operation is created merely because the binding changed before dispatch authority was obtained.

## 6. Dispatch claim integration

`DELIVER_CHILD_RESULT` uses the existing `claim_submission_dispatch` atomic authority gate.

For this operation kind, the durable stable target is `destination_logical_thread_id`; the Provider Conversation is resolved from canonical CurrentConversationBinding at claim time.

The dispatch-claim proposal supplies the resolved current tuple:

```text
(destination logical thread,
 current provider conversation,
 expected binding generation,
 current carrier,
 expected lease generation)
```

Reducer verifies the same binding / lease / single-execution invariants as ordinary submissions.

On success:

```text
SubmissionOperation.state := DISPATCHING
DispatchFence := (
  provider_conversation_ref,
  binding_generation,
  carrier_ref,
  lease_generation
)
```

Only then may Shuttle perform the one blind browser submit.

If the claim fails because the destination binding changed before commit, no provider actuation has occurred. The **same** PREPARED delivery operation may re-resolve the current parent binding and attempt a new claim.

## 7. Atomic race with parent rollover

The binding/dispatch reducer contract already provides the required serialization.

### Rollover commits before delivery dispatch claim

```text
L1: C1@g7 -> C3@g8
```

Any stale claim against C1@g7 fails. The same delivery operation resolves C3@g8 and may claim there.

### Delivery dispatch claim commits first

```text
DELIVER R17 -> DISPATCHING on C1@g7
```

The operation is now execution-owning. Parent rollover is blocked until the delivery operation reaches a non-execution-owning terminal/reconciled state.

This prevents a Review Result from being accepted into C1 while the Harness simultaneously activates C3 and then blindly inserts the same result again.

## 8. Delivery lifecycle uses SubmissionOperation lifecycle

No new parallel state vocabulary is needed:

```text
PREPARED
DISPATCHING
OBSERVED_ACCEPTED
COMPLETED
UNCERTAIN
FAILED_SAFE
CANCELLED
```

For `DELIVER_CHILD_RESULT`:

- `PREPARED`: durable result-return intent exists; no dispatch authority obtained;
- `DISPATCHING`: exact binding/lease DispatchFence committed; browser actuation may begin;
- `OBSERVED_ACCEPTED`: operational evidence proves the result-bearing user message entered the destination conversation;
- `COMPLETED`: the resulting parent Agent turn reached its normal completion boundary;
- `UNCERTAIN`: provider acceptance cannot be proven or disproven;
- `FAILED_SAFE`: provider non-acceptance is proven; same logical delivery may safely retry under policy;
- `CANCELLED`: explicit abandonment before accepted insertion.

`OBSERVED_ACCEPTED` remains execution-owning because the parent Agent may still be generating or using tools.

## 9. Durable ResultDeliveryReceipt

A delivery-specific receipt records the recipient-side transport fact without claiming semantic acceptance.

Conceptually:

```text
ResultDeliveryReceipt
- result_delivery_key
- operation_id
- result_id
- result_fingerprint
- destination_logical_thread_id
- dispatch_fence
- inserted_message_ref?        # when observable
- inserted_payload_fingerprint
- state: INSERTED | COMPLETED
- resulting_parent_turn_ref?   # when observable
- created_at
- completed_at?
```

### INSERTED

Persist when `OBSERVED_ACCEPTED` is established directly or by reconciliation.

Meaning:

> NOOS has operational proof that this logical result delivery has already been inserted into a provider conversation under the recorded DispatchFence.

Once INSERTED exists, automated replay of the same `ResultDeliveryKey` MUST NOT perform another provider insertion.

### COMPLETED

Persist when the resulting parent Agent turn reaches its completion boundary.

Meaning:

> the result-bearing turn finished transport/runtime execution.

It still does **not** mean the Designer/Human accepted the Review findings.

## 10. Crash / lost acknowledgement recovery

### Crash before dispatch claim

Operation remains PREPARED.

Recovery resolves the destination's current binding and claims dispatch normally.

### Crash after dispatch claim but before browser actuation is known

Operation is DISPATCHING.

Recovery uses the generic pre-submit baseline / conversation-head reconciliation. No new operation is created.

### Provider accepted, acknowledgement lost

Recovery proves acceptance from observable conversation evidence:

```text
expected conversation
+
message-count/head transition
+
payload fingerprint / compatible evidence
```

Then:

```text
operation -> OBSERVED_ACCEPTED
persist ResultDeliveryReceipt(state=INSERTED)
```

Do not send again.

### Still ambiguous

```text
operation -> UNCERTAIN
```

V1 pauses for manual recovery. It does not gamble by inserting the Review Result again.

## 11. Why recipient dedup is logical-thread scoped

The recipient is the parent Logical Thread, not a particular conversation.

Therefore:

```text
(R17, L1)
```

is one logical return even if L1 uses C1@g7 when the Reviewer starts and C3@g8 when the result becomes ready.

A completed delivery to L1 is not automatically redelivered after later rollover.

If future rollover bootstrap needs to preserve already-received durable Review context, that continuity is handled by normal durable bootstrap refs / current semantic documents, not by pretending the original child return is a new delivery.

## 12. Parent wait-state clearing

A child result's existence and its delivery are separate events:

```text
child RESULT_READY
→ durable delivery operation
→ INSERTED
→ parent Agent turn executes
→ COMPLETED
```

For a required Review dependency, V1 should clear the mechanical `WAIT_REVIEW(child_thread_id)` dependency when the delivery operation reaches `COMPLETED`, not merely when the child creates the Review Result.

This avoids resuming the parent while the result-bearing turn is still executing.

Clearing the mechanical wait does not set semantic acceptance and does not force `LogicalControl = CONTINUE`; the parent Agent/Human determines the next semantic boundary.

For optional maintenance notifications that do not require a parent reasoning turn, a separate control-only completion path may remain outside this contract; V1 Review Result return uses conversation delivery.

## 13. Human manual interaction

This contract prevents duplicate **Harness automation**. It cannot transactionally prevent the Human from manually pasting the same result again.

If manual interaction occurs while a delivery operation is DISPATCHING / OBSERVED_ACCEPTED / UNCERTAIN, automated recovery follows the same conservative ambiguity rule as the generic Submission Contract.

A Human may explicitly resolve an ambiguous operation, but ordinary restart must never infer non-delivery solely from a missing local callback.

## 14. Semantic acceptance remains separate

Do not introduce states such as:

```text
REVIEW_ACCEPTED
REVIEW_REJECTED
```

into ResultDeliveryReceipt.

Transport semantics are only:

```text
result exists
result inserted
result-bearing turn completed
```

After completion:

```text
Primary Design Agent reads/reasons
→ may revise / clarify / declare boundary
→ Human retains any required Freeze/Promote authority
```

NOOS does not convert Review transport success into semantic judgment.

## 15. Minimal invariants

V1 freezes the following:

1. `WorkerResult.result_id` is immutable and fingerprint-protected.
2. One `ResultDeliveryKey=(result_id,destination_logical_thread_id)` identifies one logical return.
3. ResultDeliveryKey is independent of binding/carrier generations.
4. Creating the same logical delivery is create-or-get, not create-again.
5. Concrete provider destination is resolved from canonical CurrentConversationBinding at dispatch-claim time.
6. Browser actuation requires the normal atomic `claim_submission_dispatch` permit.
7. Proven or ambiguous acceptance never triggers blind automated resend.
8. `OBSERVED_ACCEPTED` / `UNCERTAIN` remain execution-owning and block parent rollover/lease transfer under the binding contract.
9. ResultDeliveryReceipt INSERTED is the durable recipient-side applied marker for transport deduplication.
10. COMPLETED means parent turn completion, not Review acceptance.

## 16. What this closes

This contract closes the remaining V1 E2E seam identified after the corrected walkthrough:

> Review / Child Result delivery lacked an explicit idempotent logical delivery identity, create-or-get rule, recipient-side applied marker, and restart-safe no-duplicate insertion behavior.

It closes that seam by specializing the existing SubmissionOperation rather than introducing a second transport protocol.

## 17. Non-goals

This contract does not define:

- semantic Review acceptance;
- automatic Integration arbitration;
- peer Designer fan-in;
- result-content merging;
- provider-native exactly-once guarantees;
- Human browser-input exclusion;
- general event-bus delivery;
- provider DOM selectors.

## 18. Working conclusion

V1 child-result return should be understood as:

```text
immutable WorkerResult
→ one logical delivery key to parent Logical Thread
→ one specialized SubmissionOperation
→ late-bind current parent conversation
→ atomic dispatch claim
→ one blind submit
→ reconcile
→ durable INSERTED receipt
→ parent turn COMPLETED
```

The crucial rule is:

> rollover may change where an undelivered result is sent, but it must never change the identity of the delivery itself.
