# NOOS Deliberation Harness — Current Conversation Binding Contract v1

> Status: Working Design Note / V1 scope / narrow revision after v0 review
>
> Supersedes `current-conversation-binding-contract-v0.md` for binding/activation semantics.
>
> Scope: close the review-blocking seams around concrete Reducer mutation, preactivation conformance, stale-carrier lease validation, and rollover-vs-dispatch races. Review Result delivery idempotency remains a separate V1 seam.

## 1. Retained decisions from v0

The following v0 decisions remain unchanged:

1. `Current Conversation Binding != Execution Readiness`.
2. `CurrentConversationBinding(logical_thread_id, provider_conversation_ref, binding_generation)` is the single authoritative current-binding truth.
3. Forward and reverse current-binding uniqueness must both hold atomically.
4. Semantic bootstrap is scoped to a committed binding generation.
5. Fork creates a child Logical Thread; rollover preserves the Logical Thread and changes its current Provider Conversation.
6. Stale generations may be observed but cannot regain actuation authority.
7. If provider-safe preactivation is unavailable, automated activation fails closed.

This revision narrows how those decisions become executable contract.

## 2. Normative Reducer extension

Binding/lease/dispatch authority is not left as a conceptual check.

This contract normatively depends on:

`docs/harness/conversation-binding-reducer-operations-v0.md`

An implementation claiming safe V1 automated initial bind, rollover, adoption, fork/fresh child activation, or automated dispatch MUST implement formally equivalent atomic Reducer semantics for:

```text
commit_current_conversation_binding
transfer_actuation_lease
claim_submission_dispatch
```

These operations use the normal:

```text
Proposal
→ Policy
→ Authorized Delta
→ Reducer
→ ApplyResult
```

path. Ordinary runtime transitions may be machine-proposed and machine-authorized by deterministic policy; no Human approval ceremony is implied.

Direct Shuttle/content-script writes to canonical binding, lease authority, or dispatch authority are non-conforming.

## 3. Canonical current-binding relation

The only authoritative ownership relation is:

```text
CurrentConversationBinding
- logical_thread_id
- provider_conversation_ref
- binding_generation
```

Derived only:

```text
LogicalThread.active_conversation_ref
ProviderConversation.current_logical_thread_id
CURRENT_BOUND / SUPERSEDED presentation status
reverse lookup indexes
```

No derived projection may be independently mutated.

The Reducer transaction for `commit_current_conversation_binding` validates both uniqueness directions against the same tentative state:

```text
one Logical Thread -> at most one current Provider Conversation
one Provider Conversation -> at most one current Logical Thread
```

A thread-local CAS alone is non-conforming.

## 4. Canonical actuation authority

A browser carrier does not gain automated send authority from local observations alone.

The authoritative lease is:

```text
ActuationLeaseAuthority
- provider_conversation_ref
- binding_generation
- carrier_ref
- lease_generation
- status
```

Runtime attachment data may be reconstructed after restart, but automated dispatch requires a matching canonical lease.

Duplicate tabs may all observe the same conversation; only the canonical lease holder may obtain a dispatch claim.

## 5. Dispatch is claimed atomically before browser actuation

The v0 rule “verify binding + lease before dispatch” is strengthened.

A Shuttle sender MUST first obtain a successful Reducer `ApplyResult` for:

```text
claim_submission_dispatch
```

The claim atomically validates:

```text
SubmissionOperation == PREPARED
AND canonical binding == intended conversation@generation
AND canonical lease == intended carrier@lease_generation for that binding generation
AND no other unresolved execution-owning operation exists on that binding
AND runtime eligibility policy passes
```

and commits:

```text
SubmissionOperation.state = DISPATCHING
DispatchFence = (
  provider_conversation_ref,
  binding_generation,
  carrier_ref,
  lease_generation
)
```

Only then may the adapter perform the one blind provider actuation allowed by the Submission Idempotency Contract.

A local READY observation or cached lease is never itself a send permit.

## 6. Rollover and dispatch compete at one atomic boundary

The old v0 check-before-use race is explicitly closed.

For current binding:

```text
L1 -> C1@g7
```

### If dispatch claim wins first

```text
SUB-9 becomes DISPATCHING / execution-owning on C1@g7
```

Then a concurrent rollover binding mutation is rejected because the old binding has unresolved execution ownership.

### If rollover binding commit wins first

```text
L1 -> C2@g8
```

Then a dispatch claim targeting C1@g7 is rejected because canonical binding no longer matches.

Therefore both cannot validly commit.

The same pattern closes lease-transfer-vs-dispatch races:

- dispatch claim first → lease transfer blocked;
- lease transfer first → old carrier/lease-generation dispatch claim rejected.

## 7. What counts as unresolved execution ownership

Binding switch and lease transfer are blocked while an operation is in:

```text
DISPATCHING
OBSERVED_ACCEPTED
UNCERTAIN
```

`OBSERVED_ACCEPTED` is intentionally included. Provider acceptance does not mean the Agent turn or tool side effects are complete.

The old conversation must not be replaced by a new executable carrier while its accepted operation is still generating/acting.

Movement may resume after reconciliation to a non-execution-owning terminal state such as:

```text
COMPLETED
FAILED_SAFE
CANCELLED
```

If an operation remains `UNCERTAIN`, V1 fails closed.

## 8. Preactivation eligibility is a technical capability, not prompt discipline

Automated activation requires a structured preactivation eligibility receipt.

Conceptually:

```text
PreactivationEligibilityReceipt
- provider_conversation_ref
- provider_adapter_id
- adapter_capability_version
- establishment_mode
- evidence_ref
- state: ELIGIBLE
```

The adapter/policy may issue `ELIGIBLE` only under one of the modes below.

The Reducer never infers eligibility from a natural-language instruction such as “do not use tools”.

## 9. Mode A — IDENTITY_FIRST

Preferred and simplest.

Conforming sequence:

```text
create/fork/open provisional carrier
→ resolve stable provider_conversation_ref
→ no normal Agent semantic execution has occurred
→ emit ELIGIBLE receipt
→ commit binding
→ assign generation-matching actuation lease
→ perform semantic bootstrap as a normal claimed SubmissionOperation
```

This mode is conforming when provider identity is available before a normal reasoning turn is required.

## 10. Mode B — TRANSPORT_ONLY_ESTABLISHMENT

v0's Mode B is narrowed substantially.

A normal Agent turn with a prompt saying “only establish identity” is NOT a conforming quarantine.

Mode B is allowed only when the provider/adapter can technically guarantee all of the following:

1. establishment does not expose Work Item Goal/Scope, mutable artifact refs, Review material, delegation instructions, or other authority-bearing semantic payload;
2. normal tool/connector/delegation/external-write capabilities are unavailable to the establishment action by mechanism, not by model obedience;
3. the establishment action cannot mutate NOOS semantic/harness state except transport identity records explicitly allowed for establishment;
4. any establishment transcript that would otherwise become part of later reasoning context is either provider-natively excluded/quarantined from semantic context, or the provider contract proves that no normal Agent reasoning turn occurred;
5. the adapter can deterministically attest these properties in the eligibility receipt from provider capability/version, rather than guessing from output content.

If stable identity requires posting a visible normal user message and allowing a normal Assistant generation with ordinary capabilities, Mode B is unavailable.

In that provider configuration automated activation MUST fail closed.

## 11. Human-assisted adoption fallback

When neither Mode A nor conforming Mode B is available, V1 may still support explicit Human-assisted adoption.

Example:

```text
Human safely establishes/inspects provider conversation
→ Human explicitly authorizes adoption
→ stable conversation identity is known
→ commit_current_conversation_binding via Reducer
→ assign lease
→ semantic bootstrap/continuation only after committed binding
```

This fallback does not claim the pre-adoption provider history was transactionally fenced by NOOS. Human authorization accepts that existing history as the carrier being adopted.

## 12. Initial bind sequence

Safe automated initial bind:

```text
1. durable Logical Thread exists with no current binding
2. establish target conversation identity under Mode A or conforming Mode B
3. obtain PreActivationEligibilityReceipt
4. commit_current_conversation_binding: null@g0 -> C1@g1
5. assign ActuationLeaseAuthority for C1@g1
6. create semantic BOOTSTRAP SubmissionOperation = PREPARED
7. claim_submission_dispatch under C1@g1 + current lease
8. actuate once
9. reconcile/complete bootstrap operation
10. record BootstrapReceipt for C1@g1
11. only then may normal GO become eligible
```

If binding commit succeeds but later lease/bootstrap fails, the system stalls safely on C1@g1. It does not silently invent a second current conversation.

## 13. Rollover sequence

Safe automated rollover:

```text
L1 -> C1@g7

1. establish provisional C2 identity under a conforming preactivation mode
2. obtain PreActivationEligibilityReceipt(C2)
3. propose commit_current_conversation_binding expected C1@g7 -> C2@g8
4. Reducer atomically verifies:
   - expected canonical binding
   - reverse uniqueness
   - C2 eligibility
   - no execution-owning operation on C1@g7
5. commit C2@g8
6. old C1 loses actuation eligibility by binding-generation mismatch
7. assign generation-matching lease to a C2 carrier
8. create/claim/dispatch semantic bootstrap operation under C2@g8
9. record BootstrapReceipt C2@g8
10. C2 becomes continuation-eligible only when bootstrap/runtime/logical gates pass
```

If a dispatch on C1 races with step 4, exactly one of dispatch claim or binding switch may commit.

If binding commit succeeds but bootstrap fails, C2 remains canonical current but non-ready; recovery reconciles/retries bootstrap under g8. C1 is not silently reactivated.

## 14. Forked / Fresh child activation

Child Logical Thread intent is durable before provider action.

Then:

```text
child L2 PLANNED
→ establish child conversation identity under Mode A or conforming Mode B
→ initial bind null@g0 -> child C2@g1 through Reducer
→ assign lease
→ claimed semantic child bootstrap
→ BootstrapReceipt for g1
→ child execution eligible when runtime/logical gates pass
```

FORKED vs FRESH changes context provenance, not reducer safety.

If the provider's fork flow cannot expose stable child identity without a normal unfenced Agent turn, fully automated child activation is not conforming; use Human-assisted adoption or defer automation.

## 15. GO / bootstrap execution readiness

A current binding is necessary but not sufficient.

Execution eligibility conceptually requires:

```text
CurrentConversationBinding exists
AND BootstrapReceipt matches current binding generation
AND current carrier is operationally READY
AND canonical ActuationLeaseAuthority matches current binding generation/carrier
AND LogicalControl == CONTINUE (or operation-specific bootstrap eligibility)
AND no conflicting unresolved SubmissionOperation exists
```

The actual actuation still requires a successful `claim_submission_dispatch` ApplyResult.

This keeps ownership, readiness, and one-operation dispatch authority separate.

## 16. Restart / duplicate-tab recovery

Restart does not mutate CurrentConversationBinding.

Recovery reconstructs runtime attachments, then restores/transfers lease authority only through the Reducer operation.

A duplicate/reopened tab cannot send because it looks READY. It must become the canonical lease holder and then win a fresh dispatch claim.

A stale carrier possessing an old local callback/cache cannot obtain `claim_submission_dispatch` after either binding generation or lease generation has advanced.

## 17. Conformance boundary

A V1 implementation may claim:

### binding observation experiment

without implementing automated rollover/fork activation.

### safe automated binding/rollover

only if it implements equivalent semantics for:

```text
canonical CurrentConversationBinding
atomic forward/reverse uniqueness
canonical generation-scoped ActuationLeaseAuthority
commit_current_conversation_binding
transfer_actuation_lease
claim_submission_dispatch
execution-owning operation exclusion during rollover/lease transfer
Mode A or technically conforming Mode B preactivation
```

Prompt-only quarantine is non-conforming.

## 18. Review issue closure intent

This v1 specifically addresses the narrow-review findings:

- **Reducer operation gap:** binding uniqueness/commit semantics are now normatively instantiated by `commit_current_conversation_binding` in the Reducer extension.
- **Lease-holder/stale-carrier gap:** automated send requires canonical lease validation inside `claim_submission_dispatch`, not a local pre-click check.
- **Rollover/dispatch atomic race:** dispatch claim and binding/lease changes compete in one atomic Reducer authority domain; whichever commits first invalidates the other's preconditions.
- **Mode B major:** Mode B now requires technical transport-only/quarantined capability. A normal Agent turn with prompt-only restrictions is explicitly non-conforming.

This is closure intent, not an Independent Reviewer verdict.

## 19. Remaining separate V1 seam

This contract still does not define idempotent child Review Result delivery / recipient deduplication.

That remains the separate Result Delivery seam identified by the E2E walkthrough and should not be folded into binding semantics.

## 20. Working conclusion

V1 binding safety rests on three separate authority facts:

```text
1. CurrentConversationBinding says which conversation belongs to the Logical Thread.
2. ActuationLeaseAuthority says which carrier may attempt automation for that binding generation.
3. claim_submission_dispatch grants one concrete SubmissionOperation the right to begin provider actuation.
```

All three are Reducer-governed. This removes stale-carrier check/use races and prevents rollover from committing across an unresolved old-conversation execution.

Preactivation remains fail-closed unless the provider can establish identity without an unfenced normal Agent turn.
