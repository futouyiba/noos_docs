# NOOS Deliberation Harness — Command Submission / Idempotency Contract v0

> Status: Working Design Note / Not a frozen Review Candidate
>
> Purpose: define how NOOS/Shuttle should submit GO / Review / Sediment / bootstrap commands to ChatGPT Web without duplicate actuation when browser acknowledgements, page state, or extension workers are uncertain.

## 1. Core constraint

ChatGPT Web does not expose a Harness-controlled transactional submit API with an external idempotency key.

Therefore v0 should **not** claim strict exactly-once delivery.

The practical contract is:

> Persist operation identity before actuation; perform at most one blind automated dispatch; if outcome becomes uncertain, reconcile against observable conversation state before any retry.

The dangerous failure is duplicate semantic commands, not a short delay while uncertainty is resolved.

## 2. Operation identity

Every Harness-initiated actuation that can mutate a conversation or create a worker should receive a durable `operation_id` before browser action begins.

Conceptual envelope:

```text
OperationEnvelope
- operation_id
- operation_kind          # GO / REVIEW_DISPATCH / SEDIMENT / BOOTSTRAP / ...
- work_item_id
- target_logical_thread_id
- target_carrier_binding
- expected_conversation_ref when known
- payload_ref or payload_fingerprint
- created_at
```

The envelope is Harness operational state, not semantic design state.

## 3. Persist-before-actuate

Unsafe:

```text
click Send
↓
then record that GO was issued
```

A service-worker termination between those steps can cause a later retry to duplicate the command.

Preferred order:

```text
create operation_id
↓
persist operation as PREPARED
↓
verify target carrier READY
↓
actuate submit
↓
observe provider evidence
↓
advance durable operation state
```

## 4. Submission lifecycle

Candidate operational states:

```text
PREPARED
DISPATCHING
OBSERVED_ACCEPTED
COMPLETED
UNCERTAIN
FAILED_SAFE
CANCELLED
```

Approximate semantics:

- `PREPARED`: durable intent exists; no browser actuation has started;
- `DISPATCHING`: browser actuation has begun, but provider acceptance is not yet proven;
- `OBSERVED_ACCEPTED`: observable evidence shows the command entered the target conversation;
- `COMPLETED`: resulting Agent turn / worker bootstrap reached its expected runtime completion boundary;
- `UNCERTAIN`: NOOS cannot prove whether provider acceptance occurred;
- `FAILED_SAFE`: evidence indicates the provider did not accept the command and retry may be considered under policy;
- `CANCELLED`: operation intentionally abandoned before acceptance.

Do not collapse `OBSERVED_ACCEPTED` and `COMPLETED`. A prompt can be accepted while generation is still running.

## 5. Acceptance evidence

Acceptance should be inferred operationally from multiple provider probes, not semantic reasoning.

Possible evidence after a submit:

- user-message count increased relative to the recorded pre-submit baseline;
- newest user-message DOM appeared after actuation;
- newest user-message text/fingerprint matches the intended payload where appropriate;
- generation-active evidence began after submission;
- conversation route / identity remained the expected target;
- the composer cleared or transitioned through the provider's normal submit behavior.

No single probe must be treated as permanent product semantics.

The ChatGPT provider adapter may combine these signals deterministically.

## 6. Pre-submit baseline

Before actuation, record enough observable baseline to later reconcile without guessing.

Candidate baseline:

```text
conversation_ref
assistant_message_count
user_message_count
last_user_message_fingerprint
last_assistant_message_fingerprint
route_ref
observed_at
```

This baseline is especially useful when the payload is repetitive, such as `Go`.

The exact DOM fingerprint mechanism is provider-specific.

## 7. Do not blindly retry UNCERTAIN

The central safety rule:

```text
UNCERTAIN != NOT_SENT
```

If the extension worker restarts or acknowledgement is lost after clicking Send, NOOS must not infer that the command failed merely because local acknowledgement is absent.

Instead:

```text
UNCERTAIN
↓
re-attach target carrier
↓
re-read observable conversation baseline/head
↓
reconcile
```

Possible reconciliation outcomes:

```text
PROVEN_ACCEPTED
PROVEN_NOT_ACCEPTED
STILL_AMBIGUOUS
```

Only `PROVEN_NOT_ACCEPTED` may permit automatic retry under the current operation policy.

`STILL_AMBIGUOUS` should surface for manual recovery in v0 rather than risk duplicate semantic execution.

## 8. Reconciliation without semantic reasoning

Reconciliation should compare operational evidence, for example:

```text
pre-submit user message count = 17
current user message count = 18
newest user message fingerprint = expected payload
```

This is sufficient evidence that the command was accepted without asking an LLM to interpret the discussion.

For a repeated payload such as `Go`, the message-count / conversation-head transition is more useful than payload text alone.

## 9. One automated writer per carrier during actuation

To reduce ambiguity, a target carrier should have at most one active Harness submission operation at a time.

Conceptually:

```text
carrier_submission_lease(operation_id)
```

While a Harness command is `DISPATCHING` or `UNCERTAIN`, another automated Harness command must not be submitted to the same conversation.

This is an operational serialization rule, not a semantic lock on the Agent.

Human manual interaction may still occur, but simultaneous human submission during an unresolved automated dispatch can make reconciliation ambiguous. v0 may warn or temporarily disable automated continuation until the carrier is reconciled.

## 10. Do not require visible idempotency tokens in normal prompts

A tempting design is to append something like:

```text
[NOOS operation_id=abc123]
```

to every GO.

This would make reconciliation easy, but it pollutes the Agent context and exposes infrastructure protocol in normal reasoning.

v0 should avoid mandatory visible operation tokens unless empirical evidence shows DOM/head reconciliation is insufficient.

A transport-side or provider-native idempotency key would be preferable if one becomes available later.

## 11. Step Mode safety

Step Mode materially reduces the consequence of imperfect idempotency.

```text
Human triggers GO
↓
operation persisted
↓
submit once
↓
observe / reconcile
↓
stop at READY
```

There is no automatic next GO while the previous operation remains unresolved.

Therefore v0 Step Mode should prefer conservative ambiguity handling over aggressive retry.

## 12. Future Run Mode

Autonomous Run Mode raises the reliability requirement:

```text
COMPLETED operation N
AND Carrier READY
AND Logical Control CONTINUE
→ create operation N+1
```

A new operation must never be created merely because the service worker restarted or the previous local callback was lost.

The durable predecessor operation state must be reconciled first.

## 13. Fork / worker creation idempotency

Forking is more dangerous than GO because duplicate retry can create duplicate worker conversations.

Treat worker creation as its own operation:

```text
PREPARED
↓
issue fork/create action
↓
observe child tab
↓
bind provisional child carrier to operation_id
↓
bootstrap child
↓
resolve stable conversation identity
↓
COMPLETED
```

If the extension loses state after child creation, recovery should first search/reconcile against known child tab/window/provider conversation candidates associated with the operation.

Do not issue a second fork merely because the first child identity is not yet stable.

## 14. Provider conversation identity and tab identity are distinct

`tabId` identifies a browser carrier, not the durable provider conversation.

A reload can preserve conversation identity while changing content-script lifecycle; a fork may create a tab before the provider conversation identity exists.

Submission records should therefore retain both when available:

```text
carrier_ref = windowId/tabId
provider_conversation_ref = stable conversation identity
```

Reconciliation should prefer provider conversation identity once stable, but may use browser carrier identity during ATTACHING / STABILIZING.

## 15. Failure policy

For ordinary v0 commands, choose the safer failure direction:

- false `not sent` followed by retry can duplicate reasoning;
- false `sent` usually causes a visible stall that a human can recover.

Therefore:

> When acceptance cannot be established, pause rather than resend.

This is especially appropriate in Step Mode.

## 16. Minimal operation record

A v0 operation record may remain small:

```text
operation_id
operation_kind
work_item_id
logical_thread_id
target_carrier_ref
provider_conversation_ref
payload_fingerprint
pre_submit_baseline
state
created_at
last_observed_at
resulting_turn_ref when observable
```

This is enough to support restart-safe dispatch/reconciliation without storing deliberation semantics.

## 17. Relationship to Carrier Runtime State

Submission lifecycle and carrier runtime are related but distinct.

Examples:

```text
operation PREPARED + carrier READY
→ dispatch allowed

operation DISPATCHING + carrier GENERATING
→ likely accepted, continue observation

operation UNCERTAIN + carrier READY
→ reconcile before any new command

operation COMPLETED + carrier READY
→ eligible for later continuation if Logical Control permits
```

Do not derive operation completion solely from carrier READY; the READY state may arise after an unrelated reload or failed submit.

## 18. Current working conclusion

The first Shuttle should provide restart-safe **at-most-one blind dispatch per operation**, not pretend to provide transactional exactly-once delivery.

Durable operation identity + persist-before-actuate + observable reconciliation + conservative retry policy is sufficient for the first Step Mode slice.

The next design question is how to model and recover logical conversation identity across fork, reload, tab replacement, and long-lived multi-tab sessions without letting transient browser `tabId` become the identity of the Design/Review/Sedimentation thread.
