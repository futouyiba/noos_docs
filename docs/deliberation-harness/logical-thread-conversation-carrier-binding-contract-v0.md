# NOOS Deliberation Harness — Logical Thread / Provider Conversation / Browser Carrier Binding Contract v0

> Status: Working Design Note / Not a frozen Review Candidate
>
> Purpose: define durable identity and rebinding semantics across NOOS Logical Threads, provider conversations, and browser tabs so that reload, fork, rollover, recovery, and multi-tab execution do not create ambiguous or double-active reasoning carriers.

## 1. Core distinction

The Harness must not collapse three different identities:

```text
Logical Thread
≠ Provider Conversation
≠ Browser Carrier
```

Recommended ownership:

- **Logical Thread** — durable NOOS identity for one continuing reasoning role/workstream;
- **Provider Conversation** — provider-owned conversation/session that currently carries the reasoning context;
- **Browser Carrier** — tab/window execution surface currently attached to a provider conversation.

The Logical Thread is the stable identity. Conversations and tabs are replaceable carriers.

## 2. Logical Thread

A Logical Thread represents continuity of one reasoning role under one Work Item and scope lineage.

Examples:

- Design Thread for Work Item X;
- Independent Reviewer Thread for Review Target Y;
- Sedimentation Thread forked from Design Thread X;
- Integration Thread for a specific review/design reconciliation.

A Logical Thread survives:

- page reload;
- tab close/reopen;
- provider conversation rollover;
- extension service-worker restart;
- browser restart, provided the provider conversation can be recovered/rebound.

Minimal durable identity:

```text
logical_thread_id
work_item_id
role
parent_thread_id?   # for forked worker relations
status
active_conversation_binding?
```

The Logical Thread should not embed detailed semantic state.

## 3. Provider Conversation

A Provider Conversation is a concrete provider-side reasoning container, such as one ChatGPT conversation URL/id.

It may be:

- active carrier of a Logical Thread;
- historical/superseded carrier after rollover;
- child conversation created by fork;
- orphan/unbound until NOOS resolves its relation.

Important rule:

> A Provider Conversation may carry at most one active Logical Thread binding at a time.

The same Logical Thread may have multiple historical Provider Conversations across its lifetime, but only one may be authoritative as the current active carrier.

Conceptual history:

```text
LogicalThread L1
  ├─ Conversation C1   # historical
  ├─ Conversation C2   # historical
  └─ Conversation C3   # active
```

## 4. Browser Carrier

A Browser Carrier is a transient execution surface, usually:

```text
(windowId, tabId)
```

It is not durable identity.

The same Provider Conversation may appear in:

- one tab today;
- a different tab after reopening;
- multiple duplicate tabs accidentally.

Therefore:

> tab identity must never be used as Logical Thread identity or Provider Conversation identity.

Browser carriers may be rebound after reload/reopen based on recovered provider conversation identity.

## 5. Binding layers

Recommended binding chain:

```text
Logical Thread
   ↓ active_conversation_binding
Provider Conversation
   ↓ current_execution_attachment
Browser Carrier
```

These bindings have different authority.

### 5.1 Logical Thread -> Provider Conversation

This is durable Harness state.

It decides which provider conversation is the current continuation carrier for the Logical Thread.

### 5.2 Provider Conversation -> Browser Carrier

This is runtime attachment state.

It says which tab currently exposes the provider conversation for observation/actuation.

Reload/reopen may change this attachment without changing the Logical Thread binding.

## 6. Single-active-conversation invariant

For one Logical Thread:

```text
at most one Provider Conversation is ACTIVE at a time
```

This is the main anti-double-active rule.

A rollover must not become:

```text
L1 -> C1 active
L1 -> C2 active
```

Instead:

```text
before:
L1 -> C1 ACTIVE

rollover transition:
C2 created + bootstrapped + stabilized

commit binding switch:
L1.active = C2
C1 = SUPERSEDED/HISTORICAL
```

The binding switch should be explicit and atomic from NOOS's perspective.

## 7. Rollover semantics

Rollover means:

> same Logical Thread, new Provider Conversation.

It does **not** create a new Work Item or new reasoning identity.

Conceptual lifecycle:

```text
L1 -> C1 ACTIVE
↓ context pressure / recovery reason
create C2
↓
bootstrap Role + Work Item + Goal + durable refs + handoff context
↓
C2 stabilizes
↓
validate continuation readiness
↓
commit active binding switch
↓
L1 -> C2 ACTIVE
C1 -> SUPERSEDED
```

Until the switch commits, C1 remains the authoritative active carrier.

If C2 creation/bootstrap fails, L1 remains on C1.

## 8. Fork semantics

Fork is different from rollover.

Fork means:

> create a new Logical Thread whose initial Provider Conversation inherits context from a parent conversation.

Examples:

- Design -> Reviewer;
- Design -> Sedimentation;
- Design -> Research worker.

Conceptual lifecycle:

```text
Parent Logical Thread L1
  active conversation C1

fork
↓
new conversation C2
↓
new Logical Thread L2
  parent_thread_id = L1
  active conversation = C2
```

Therefore:

```text
fork creates a new Logical Thread
rollover preserves the same Logical Thread
```

This distinction should be explicit in the Harness model.

## 9. Reload / reopen semantics

Reload/reopen should normally change only Browser Carrier attachment.

Example:

```text
L1 -> C3 ACTIVE
C3 -> tab 101

user closes tab 101

later opens C3 in tab 207

L1 -> C3 ACTIVE        # unchanged
C3 -> tab 207          # new attachment
```

No Logical Thread migration occurs.

## 10. Duplicate tabs

The same Provider Conversation may accidentally be open in multiple tabs.

Example:

```text
C3 -> tab 207
C3 -> tab 233
```

This is not automatically a semantic conflict, but it is an execution hazard because two tabs could submit concurrently.

Recommended v0 rule:

> one Provider Conversation may have multiple observed carriers, but only one may hold the actuation lease.

Conceptual:

```text
C3
  observed_carriers = {207, 233}
  active_actuation_carrier = 207
```

Other duplicate tabs are read-only from Harness automation unless the lease is explicitly transferred.

## 11. Actuation lease

The actuation lease prevents duplicate tabs or restarted workers from both sending commands.

Minimal semantics:

```text
conversation_ref
actuation_carrier
lease_epoch / generation
```

Only the current lease holder may perform automated submission.

If the active tab disappears:

```text
lease invalidated
↓
reconcile duplicate/reopened carriers
↓
assign new lease
```

This is an operational lease, not semantic authority.

## 12. Binding version / epoch

Because rollover and carrier reassignment can race with delayed events, the active binding should carry a monotonically changing epoch/version.

Example:

```text
L1 binding_epoch = 7
active conversation = C3
```

After rollover:

```text
L1 binding_epoch = 8
active conversation = C4
```

An event arriving from C3 carrying epoch 7 after the switch must not mutate the current active binding.

This is especially important for delayed browser messages or service-worker restart recovery.

The exact storage mechanism is implementation detail; the invariant is semantic:

> stale carrier events must not reactivate a superseded conversation.

## 13. Conversation status

Provider Conversation lifecycle need not be elaborate.

Candidate operational states:

```text
PROVISIONAL
ACTIVE
SUPERSEDED
ORPHANED
CLOSED/UNAVAILABLE
```

- `PROVISIONAL`: newly created/forked/rollover conversation not yet committed to a Logical Thread as active;
- `ACTIVE`: current continuation carrier for one Logical Thread;
- `SUPERSEDED`: historical carrier after rollover;
- `ORPHANED`: discovered conversation with uncertain/no resolved Harness binding;
- `CLOSED/UNAVAILABLE`: provider carrier no longer usable.

These are execution states, not semantic judgments about the work.

## 14. Rebinding after restart

After browser or extension restart, NOOS should reconstruct bindings from durable state rather than assume tabs are authoritative.

Conceptual recovery:

```text
load durable Logical Thread bindings
↓
observe currently open ChatGPT tabs
↓
resolve provider conversation identities
↓
match tabs to known conversations
↓
restore runtime attachments
↓
reissue actuation leases where safe
```

Unknown tabs may remain unbound until explicitly adopted.

Known active conversations with no open tab may remain logically active but operationally unattached until reopened.

## 15. Do not infer Logical Thread identity from title alone

Conversation titles are useful human labels but weak identity.

Reasons:

- titles can change;
- duplicate titles are possible;
- fork-generated titles may be provisional;
- user renaming may occur;
- provider title generation is not guaranteed to be stable.

Title should be metadata/search aid, never authoritative identity.

Provider conversation id/ref/URL identity is preferred where stable.

## 16. Parent/child lineage

For Harness workers, record parent-child relation at the Logical Thread layer, not only at the browser or conversation layer.

Example:

```text
Design L1
├─ Reviewer L2
├─ Sedimentation L3
└─ Evidence Worker L4
```

The provider conversations implementing these workers may later rollover independently.

This lets NOOS answer operational questions such as:

- which Reviewer result belongs back to which Design Thread?
- which Sedimentation worker was forked from which design state?
- which worker is historical vs active?

without interpreting the semantic content of the conversations.

## 17. Return routing

When a child worker finishes, its result should route to the parent Logical Thread, not to whatever browser tab currently happens to be active.

Conceptually:

```text
child Logical Thread L2
parent_thread_id = L1
↓
result/receipt
↓
resolve L1.active_conversation_binding
↓
deliver to current active carrier of L1
```

This remains correct even if L1 rolled over from C1 to C5 while the Reviewer was running.

This is a major reason to route by Logical Thread identity instead of tab identity.

## 18. Race example: Review running during Design rollover

Initial:

```text
Design L1 -> C1
Reviewer L2 -> C2
L2.parent = L1
```

Design rolls over while Reviewer works:

```text
L1 -> C3
C1 -> SUPERSEDED
```

Reviewer completes.

Correct routing:

```text
L2 result
→ parent L1
→ current active conversation C3
```

Incorrect routing would send the result back to C1 merely because C1 was the parent browser tab at fork time.

## 19. Human-visible naming vs identity

Human labels may still be generated for usability:

```text
Design · Capture Harness
Review · Capture Harness · 2026-08-30
沉淀 · Capture Harness · 2026-08-30 23:44
```

But these names are presentation metadata layered over durable ids.

## 20. Minimal durable records for v0

A small model may be sufficient:

```text
LogicalThread
- logical_thread_id
- work_item_id
- role
- parent_thread_id?
- active_conversation_ref?
- binding_epoch
- status

ProviderConversation
- conversation_ref
- provider
- logical_thread_id?
- lifecycle_status
- created_from_conversation_ref?

CarrierAttachment
- conversation_ref
- window_id
- tab_id
- runtime_state
- actuation_lease?
```

This is conceptual, not a database schema commitment.

## 21. What NOOS should not do

Do not:

- use tabId as durable thread identity;
- use conversation title as authoritative identity;
- allow two ACTIVE conversations for one Logical Thread;
- resend child results to the original parent tab without resolving the current parent Logical Thread binding;
- treat rollover as a new Work Item;
- treat fork as merely another tab of the same Logical Thread;
- allow duplicate tabs to actuate the same provider conversation concurrently;
- let stale delayed events reactivate superseded bindings.

## 22. Current working conclusion

The key identity hierarchy is:

```text
Logical Thread = durable reasoning-workstream identity
Provider Conversation = replaceable context carrier
Browser Carrier = replaceable execution surface
```

The active Logical Thread -> Provider Conversation binding must be single-active and versioned. Browser carrier attachments are recoverable runtime state. Fork creates a new Logical Thread; rollover preserves the existing Logical Thread while switching its active Provider Conversation.

The next design question is how Work Item, Logical Thread, and Goal/Scope control relate: whether one Work Item can own multiple peer Design Threads, how role-specific child threads inherit scope, and which control fields are inherited vs re-declared at fork/bootstrap.
