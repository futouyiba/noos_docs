# NOOS Deliberation Harness — Primary Design + Child Worker Lifecycle v0

> Status: Working Design Note / V1 scope / Not a frozen Review Candidate
>
> Purpose: define the minimal V1 lifecycle for one Primary Design Logical Thread plus temporary child workers such as Reviewer, Sedimentation, and Evidence workers.

## 1. V1 topology

V1 intentionally supports one Primary Design Logical Thread per Work Item.

```text
Work Item X
└── Primary Design Thread L1
    ├── Reviewer L2
    ├── Sedimentation L3
    ├── Evidence Worker L4
    └── later Reviewer L5
```

Child workers are supporting threads, not peer designers.

V1 does not support multiple concurrent peer Design Threads that can independently own or mutate the semantic head.

## 2. Core ownership rule

The Primary Design Thread owns the continuing semantic trajectory of the Work Item.

Child workers may:

- inspect scoped material;
- produce review, sedimentation, evidence, or other bounded results;
- write explicitly authorized supporting artifacts;
- return results to the Primary Design Thread.

Child workers do not autonomously replace the Primary Design Thread's semantic authority or continuation state.

## 3. Lifecycle states

A child Logical Thread may use the following minimal lifecycle:

```text
PLANNED
SPAWNING
BOOTSTRAPPING
ACTIVE
RESULT_READY
RETURNING
COMPLETED
RETIRED
```

Operational failure states may include:

```text
SPAWN_UNCERTAIN
BROKEN
CANCELLED
```

These are Harness lifecycle states, not semantic judgments about the quality of the worker result.

## 4. PLANNED

A child worker exists as a durable Harness intent before any browser action occurs.

Minimum intent fields:

```text
child_thread_id
parent_thread_id
work_item_id
role
creation_mode: FORKED | FRESH
operation_goal
operation_scope
return_route
relevant_artifact_refs
```

Persisting the intent first allows fork/spawn execution to recover after browser or extension interruption.

## 5. SPAWNING

NOOS executes the provider/browser action that creates the worker carrier.

Examples:

- fork current Design conversation for Sedimentation;
- create a fresh ChatGPT conversation for Independent Review;
- open a new tab and bootstrap a dedicated Evidence worker.

Spawn is itself an idempotency-sensitive operation. If the result is uncertain, reconcile before attempting another spawn.

Do not equate `tab created` with `worker created`.

## 6. BOOTSTRAPPING

After the provider conversation/carrier exists, NOOS establishes the worker contract.

### 6.1 Inherited Work Item context

All child workers inherit:

- `work_item_id`;
- Primary Goal;
- durable Work Item Scope / Non-goals;
- parent Logical Thread identity;
- return route;
- artifact references needed by the worker.

### 6.2 Specialized operation contract

Each worker receives a role-specific Operation Goal and Operation Scope.

Example Reviewer:

```text
Role: Independent Reviewer
Operation Goal: review exact frozen target R17
Operation Scope: report findings only; do not redesign outside target
```

Example Sedimentation worker:

```text
Role: Sedimentation / Memory Curator
Operation Goal: preserve missing durable reasoning from inherited history
Operation Scope: do not continue the main design trajectory
```

Task-level Goal is inherited; worker-level Operation Goal is specialized.

## 7. FORKED vs FRESH creation

### FORKED

Use when inherited conversational history is valuable and does not undermine the worker role.

Primary V1 use:

- Sedimentation.

Advantages:

- cheap high-fidelity context transport;
- no need to reconstruct long deliberation history;
- branch reasoning remains isolated from the future main Design context.

### FRESH

Use when independence from the Design conversation matters.

Primary V1 use:

- Independent Review.

Bootstrap from explicit durable inputs such as:

- Work Item Goal / Scope;
- exact frozen target;
- Reviewer contract;
- selected evidence refs.

Do not inherit the Designer's full conversational reasoning trajectory by default.

## 8. ACTIVE

The child Agent performs its role autonomously inside its bounded Operation Goal / Scope.

NOOS manages:

- browser carrier runtime;
- submission safety;
- Goal/Scope anchoring when needed;
- rollover/recovery if the worker conversation itself needs continuation;
- explicit external blockers.

NOOS does not direct the worker's detailed reasoning path.

## 9. RESULT_READY

A worker reaches `RESULT_READY` when it has produced the durable or returnable output required by its operation contract.

Examples:

- Reviewer: Review Report / findings artifact exists;
- Sedimentation: target durable documents were updated and omission pass completed;
- Evidence worker: evidence package / source synthesis is available.

`RESULT_READY` means a result exists, not that the parent has semantically accepted it.

## 10. Result form

Prefer a durable result reference when the result matters beyond immediate conversation transport.

Conceptually:

```text
WorkerResult
- child_thread_id
- parent_thread_id
- work_item_id
- result_ref or return_payload
- relevant revision/ref
- completion receipt
```

NOOS need not understand the semantic contents of the result.

## 11. RETURNING

Results route to the parent Logical Thread, not to the browser tab or provider conversation that originally created the worker.

```text
child L2 result
→ resolve parent L1
→ resolve L1 current ACTIVE Provider Conversation
→ deliver result there
```

This remains correct even if the parent rolled over while the child was working.

Do not persist return routes as raw `tabId` destinations.

## 12. Parent continuation after return

A child result does not automatically determine the parent's semantic next step.

Examples:

Reviewer result:

```text
Review result returned to Primary Design Thread
→ Designer reads/reasons
→ Designer decides revise / clarify / declare next boundary
```

Sedimentation completion:

```text
Sedimentation complete
→ durable docs updated
→ Primary Design Thread remains on its existing trajectory
```

NOOS should not convert child semantic conclusions into parent decisions by itself.

## 13. Parent wait states

The parent may enter a logical wait state while a child is active.

Examples:

```text
WAIT_REVIEW(child_thread_id=L2)
WAIT_WORKER(child_thread_id=L3)
```

When the expected result is returned, the Harness may clear the mechanical wait and deliver the result, but should not infer semantic acceptance.

For Step Mode, the parent may then await Human/explicit `GO` after the Agent receives the result.

## 14. Multiple child workers

V1 may allow multiple supporting children to exist over the lifetime of a Work Item.

Concurrency should remain conservative.

Safe/common:

```text
Primary Design waits for one Independent Reviewer
Sedimentation may run as a separate bounded maintenance worker
```

V1 does not need a general dependency DAG or arbitrary fan-in/fan-out scheduler.

Each child has one explicit parent and one explicit return route.

## 15. COMPLETED vs RETIRED

### COMPLETED

The worker operation finished and its result was returned or durably recorded.

The provider conversation may still exist for audit/history.

### RETIRED

The worker is no longer an active execution participant.

Retirement may:

- release the active actuation lease;
- allow normal tab discard/closure;
- preserve Logical Thread and conversation refs for provenance;
- remove the worker from active scheduling.

Retire the worker, not its history.

## 16. Failure / recovery

### Spawn uncertainty

If a fork/new-conversation action may have succeeded but acknowledgement was lost:

```text
SPAWN_UNCERTAIN
→ reconcile existing tabs/conversations
→ bind recovered child if found
→ only create another child if non-creation is proven
```

### Active carrier failure

If a worker tab is discarded/reloaded/broken:

```text
Logical Thread remains valid
→ recover or replace Browser Carrier
```

If provider conversation is unrecoverable:

- for a restartable bounded worker, bootstrap a replacement conversation under the same child Logical Thread;
- preserve previous conversation as historical/superseded;
- do not silently create a second child Logical Thread for the same operation.

## 17. Rollover inside a child worker

A child worker may itself need rollover if it becomes long-running.

Rollover preserves child Logical Thread identity:

```text
Reviewer L2
C2 ACTIVE
→ rollover
C2 SUPERSEDED
C5 ACTIVE
```

Parent/return routing still points to `L2`, not to a specific provider conversation.

## 18. Human correction scope

A Human correction inside a child worker is thread-local unless the Human explicitly changes Work Item-level Goal/Scope.

Examples:

- "Reviewer: only check the five previously reported issues" → Reviewer operation-scope correction;
- "This entire Work Item must no longer consider provider-specific document mutation" → explicit Work Item Scope change.

NOOS should not infer a Work Item-level scope mutation from ordinary conversational correction.

## 19. Minimal V1 invariants

1. One Primary Design Logical Thread per Work Item.
2. Every child worker has exactly one parent Logical Thread.
3. Every child worker has an explicit role, Operation Goal, and Operation Scope.
4. Forked vs Fresh creation is explicit.
5. Child results return by Logical Thread routing, not raw tab routing.
6. Child semantic results do not automatically mutate parent semantic decisions.
7. Child spawn is durable/idempotency-aware.
8. Worker history survives retirement for provenance.
9. Rollover replaces Provider Conversation but preserves Logical Thread.
10. V1 does not support peer Design authority or general agent graphs.

## 20. V1 happy paths

### Sedimentation

```text
Design L1 ACTIVE
→ plan Sedimentation L2 (FORKED)
→ fork current conversation
→ stabilize/bootstrap child
→ L2 ACTIVE
→ write Current-adjacent durable memory
→ RESULT_READY
→ return completion/result refs to L1 if needed
→ COMPLETED
→ RETIRED
```

The Design Thread does not inherit Sedimentation scratch reasoning.

### Independent Review

```text
Design L1 reaches review boundary
→ parent WAIT_REVIEW
→ plan Reviewer L2 (FRESH)
→ create fresh conversation
→ bootstrap with frozen target + review contract
→ L2 ACTIVE
→ Review Report created
→ RESULT_READY
→ route report to current active conversation of L1
→ L2 COMPLETED / RETIRED
→ L1 receives report and reasons about it
```

## 21. Explicit V2 deferrals

Defer:

- multiple peer Design Threads;
- competing Candidate branches;
- automatic Integration arbitration;
- general dependency DAGs;
- multi-parent workers;
- automatic semantic result merging;
- worker marketplace / scheduling optimization.

## 22. Working conclusion

V1 should model a simple reasoning spine:

> one Primary Design Thread, plus bounded child workers with explicit roles, explicit creation mode, durable return routing, and clear retirement.

This directly matches the proven manual workflow while leaving semantic reasoning with the agents rather than the Harness.
