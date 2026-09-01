# NOOS Deliberation Harness — Current Conversation Binding Contract v0

> Status: Working Design Note / V1 scope / Not a frozen Review Candidate
>
> Purpose: close the V1 identity/activation seams around initial bind, rollover, adoption, forked/fresh child activation, reverse uniqueness, and pre-activation execution safety.

## 1. Core decision

The Harness should distinguish:

```text
Current Conversation Binding
!=
Execution Readiness
```

The canonical identity question is:

> Which Provider Conversation is the current continuation carrier for this Logical Thread?

The runtime-readiness question is separate:

> Is that current conversation bootstrapped, attached to a READY carrier, logically allowed to continue, and free of unresolved actuation?

Do not overload one `ACTIVE` flag with both meanings.

## 2. Canonical source of truth

V1 declares one authoritative relation:

```text
CurrentConversationBinding
- logical_thread_id
- provider_conversation_ref
- binding_generation
```

This relation is the single source of truth for current Logical Thread -> Provider Conversation ownership.

Everything else is derived or historical:

- `LogicalThread.active_conversation_ref` is a projection of CurrentConversationBinding;
- reverse lookup `ProviderConversation.current_logical_thread_id` is a derived/indexed view;
- conversation `ACTIVE/SUPERSEDED` presentation status is derived from current binding plus binding history;
- historical conversation ownership is lineage/provenance, not a second current-binding truth.

No runtime adapter, Shuttle event handler, or child-worker lifecycle code may independently write competing current-binding fields.

## 3. V1 uniqueness invariants

At any committed generation:

```text
one Logical Thread -> at most one current Provider Conversation
one Provider Conversation -> at most one current Logical Thread
```

Both directions must be enforced by one atomic reducer transition.

A thread-local compare-and-set is insufficient because concurrent transitions such as:

```text
L1 -> C7
L2 -> C7
```

must not both succeed.

## 4. Binding mutation is an operational authority transition

Initial bind, rollover switch, recovery adoption, and explicit human adoption all mutate durable Harness state.

They therefore use the normal state-mutation authority path:

```text
Proposal
-> deterministic Policy
-> Authorized Delta
-> Reducer
-> ApplyResult
```

This does not imply Human approval for ordinary runtime coordination.

For V1:

- initial bind may be machine-authorized after an explicitly created/activated Work Item and resolved target conversation;
- rollover may be machine-authorized when deterministic preconditions pass;
- restart/recovery rebinding to the already-known current conversation is reconstruction, not a semantic authority event;
- adoption of an arbitrary pre-existing conversation that is not already part of the thread lineage requires explicit Human authorization.

The key rule is:

> Runtime coordination may propose; only the reducer commits the canonical binding.

## 5. Atomic compare-and-set transition

Conceptual proposal:

```text
ConversationBindingProposal
- logical_thread_id
- expected_current_conversation_ref?   # null for initial bind
- expected_binding_generation
- target_conversation_ref
- reason: INITIAL | ROLLOVER | ADOPT | RECOVERY_REPAIR
- source_operation_id
```

Deterministic policy/reducer preconditions include at minimum:

```text
1. canonical current binding for logical_thread_id
   matches expected_current_conversation_ref + expected_binding_generation

2. target_conversation_ref has stable provider identity

3. target_conversation_ref is not the current conversation of another Logical Thread

4. no stale binding generation is being committed

5. any required actuation on the old/current carrier is not DISPATCHING or UNCERTAIN

6. target satisfies the provider-specific pre-activation eligibility contract
```

On success, one reducer transaction:

```text
write CurrentConversationBinding(
  logical_thread_id,
  target_conversation_ref,
  expected_binding_generation + 1
)
```

and returns an ApplyResult containing the committed generation.

The old conversation becomes historical/superseded as a derived consequence.

If any precondition fails, no partial binding mutation occurs.

## 6. Binding generation

`binding_generation` monotonically increases whenever the canonical current conversation changes.

Example:

```text
L1 -> C1 @ generation 7

rollover

L1 -> C3 @ generation 8
```

Delayed events from generation 7 may be observed for audit/reconciliation but cannot mutate generation 8 state.

The binding generation is also part of runtime actuation fencing.

## 7. Carrier lease must bind to binding generation

A Browser Carrier may actuate only when its lease matches the canonical current binding generation.

Conceptually:

```text
ActuationLease
- provider_conversation_ref
- carrier_ref
- lease_generation
- binding_generation
```

Before automated dispatch, Shuttle verifies:

```text
lease.provider_conversation_ref
    == CurrentConversationBinding.provider_conversation_ref

AND
lease.binding_generation
    == CurrentConversationBinding.binding_generation
```

This prevents an old tab from sending after rollover even if it remains visually READY.

## 8. Binding does not imply continuation readiness

After the canonical binding switches to a new conversation, that conversation may still require semantic bootstrap.

Therefore GO eligibility is stronger than merely having a current binding.

V1 GO eligibility becomes conceptually:

```text
CurrentConversationBinding exists
AND bootstrap_completed_for_generation == current binding_generation
AND Carrier == READY
AND current carrier holds valid actuation lease for that binding_generation
AND LogicalControl == CONTINUE
AND no unresolved required SubmissionOperation exists
```

This avoids inventing a second meaning for `ACTIVE`.

## 9. Bootstrap completion is generation-scoped

Semantic bootstrap completion is recorded against the binding generation it initialized.

Conceptually:

```text
BootstrapReceipt
- logical_thread_id
- provider_conversation_ref
- binding_generation
- bootstrap_operation_id
- state: COMPLETED
```

A receipt from an old generation cannot make a new binding continuation-ready.

Rollover therefore cannot accidentally inherit `bootstrap complete` from the superseded conversation.

## 10. Pre-activation provider problem

Some providers may expose a stable new conversation identity before any Agent generation.

Others, including some ChatGPT Web flows, may not expose a stable conversation identity until after a first message or fork stabilization step.

This creates a real safety issue:

```text
old conversation still current
+
new provisional conversation executes meaningful Agent work
```

could produce two side-effect-capable reasoning carriers before the binding switch commits.

Prompt text such as `do not use tools` is not a sufficient authority boundary by itself.

## 11. Conforming pre-activation modes

Automated initial bind / rollover / child activation is allowed only if the provider adapter supports at least one conforming mode.

### Mode A — IDENTITY_FIRST

Preferred.

```text
create/fork/open provisional carrier
-> resolve stable provider_conversation_ref
-> no Agent work has executed
-> commit CurrentConversationBinding
-> run semantic bootstrap
```

### Mode B — QUARANTINED_ESTABLISHMENT

If stable conversation identity requires an initial provider turn, the adapter must provide an establishment step whose externally meaningful side effects are fenced.

The establishment operation may only establish provider identity / transport readiness. It must not carry normal Work Item instructions, mutable artifact refs, delegated work, or semantic execution commands.

Only after stable identity is resolved does NOOS commit CurrentConversationBinding and send the real semantic bootstrap.

A fixed prompt instruction alone is not the fencing mechanism; the adapter/policy must restrict what pre-activation operation can be issued and what authority-bearing payload/capabilities are exposed.

### No conforming mode

If neither IDENTITY_FIRST nor a genuinely quarantined establishment path is available:

```text
automated activation fails closed
```

The Harness may fall back to explicit Human-assisted adoption after the conversation is safely established and inspected.

V1 must not pretend automated fork/rollover is implementation-safe in this provider configuration.

## 12. Important simplification: no speculative semantic bootstrap

Do not perform the normal Role/Goal/Scope/artifact bootstrap before binding commit merely to discover the conversation identity.

Preferred ordering:

```text
prepare provisional carrier
-> establish stable provider identity under a conforming pre-activation mode
-> atomic binding commit
-> issue semantic bootstrap for committed generation
-> record BootstrapReceipt
-> continuation becomes eligible
```

This cleanly separates:

- transport establishment;
- durable ownership;
- semantic initialization;
- normal execution.

## 13. Initial bind

Initial bind is the same transition with no previous conversation:

```text
expected_current = null
expected_generation = 0
-> target C1
-> commit generation 1
-> semantic bootstrap C1@g1
-> BootstrapReceipt completed
-> GO eligible when runtime/logical gates also pass
```

No separate identity model is needed for the Primary Design Thread bootstrap.

## 14. Rollover

Rollover preserves Logical Thread identity and changes only its current conversation binding.

Safe V1 sequence:

```text
L1 -> C1 @ g7

1. create provisional C2
2. establish stable C2 identity using conforming pre-activation mode
3. ensure C1 has no DISPATCHING / UNCERTAIN required actuation
4. propose CAS: expected C1@g7 -> C2@g8
5. reducer commits atomically
6. old C1 loses actuation eligibility through binding-generation fencing
7. semantic bootstrap C2@g8
8. record BootstrapReceipt for g8
9. C2 may become GO-eligible
```

If steps 1–4 fail, C1 remains canonical current.

If binding commit succeeds but semantic bootstrap fails, C2 remains canonical current but not continuation-ready. Recovery retries/reconciles bootstrap under generation 8; it does not silently reactivate C1.

A rollback, if ever supported, is another explicit binding transition with a new generation, not mutation of history.

## 15. Forked/Fresh child activation

Child creation still follows the Child Worker Lifecycle contract.

The child Logical Thread is durably PLANNED before provider action.

Then:

```text
child L2 PLANNED
-> create provisional child conversation C2
-> establish C2 identity under conforming pre-activation mode
-> initial bind L2: null@g0 -> C2@g1
-> semantic child bootstrap
-> BootstrapReceipt L2/C2/g1
-> child ACTIVE / execution-eligible when runtime gates pass
```

This applies to both:

- FORKED Sedimentation;
- FRESH Independent Reviewer.

FORKED vs FRESH controls context provenance, not binding correctness.

## 16. Adoption

Adoption binds an already-existing provider conversation to a Logical Thread.

Because an arbitrary existing conversation may contain unknown semantic history and may already belong to another workflow, V1 requires explicit Human authorization unless the conversation is already a known historical carrier of the same Logical Thread.

The reducer still enforces reverse uniqueness and expected-generation CAS.

Adoption does not rewrite the conversation's prior history; it only establishes future Harness ownership/actuation.

## 17. Restart / recovery

Service-worker/browser restart does not itself change CurrentConversationBinding.

Recovery:

```text
load canonical CurrentConversationBinding
-> observe tabs
-> resolve provider identities
-> attach matching carriers
-> restore/transfer actuation lease using current binding_generation
-> reconcile unresolved SubmissionOperations
```

A visible tab does not become current merely because it exists.

A stale historical conversation cannot be promoted by a delayed browser event.

## 18. Duplicate tabs

Multiple Browser Carriers may expose the same canonical current Provider Conversation.

This does not create multiple bindings.

Only one carrier may hold the generation-matching actuation lease.

The other carriers are observation-only for Harness automation.

## 19. Derived conversation status

Presentation/runtime status may show:

```text
PROVISIONAL
CURRENT_BOUND
SUPERSEDED
ORPHANED
UNAVAILABLE
```

But these are not independently authoritative current-binding fields.

For example:

```text
CURRENT_BOUND
```

is derived from CurrentConversationBinding.

A conversation can be current-bound but not yet bootstrap-complete / continuation-ready.

## 20. What this contract closes

This contract closes the four binding/activation gaps identified by the corrected V1 E2E walkthrough:

### GAP-1
Canonical binding mutation now uses the normal Proposal/Policy/Authorized Delta/Reducer path.

### GAP-2
Automated activation requires identity-first or genuinely quarantined pre-activation establishment; normal semantic bootstrap is forbidden before binding commit.

### GAP-3
Forward and reverse uniqueness are checked atomically by one reducer transition.

### GAP-4
CurrentConversationBinding is the single source of truth; other active/current fields are projections or lineage.

## 21. Non-goals

This contract does not define:

- semantic supervisor behavior;
- multiple peer Designer authority;
- general Agent DAGs;
- Review Result delivery idempotency;
- provider-specific DOM selectors;
- database schema/storage engine;
- Human approval for ordinary rollover runtime coordination.

Review Result delivery idempotency remains a separate V1 transport seam.

## 22. Working conclusion

V1 should treat conversation ownership as one small, authoritative, versioned relation:

```text
CurrentConversationBinding
= canonical Logical Thread -> current Provider Conversation relation
```

The reducer is the only component that commits binding changes. Reverse uniqueness is atomic. Runtime actuation is fenced by binding generation. Semantic bootstrap is generation-scoped and occurs only after ownership commits. If a provider cannot establish a stable conversation safely before semantic execution, automated activation fails closed instead of relying on prompt-only discipline.
