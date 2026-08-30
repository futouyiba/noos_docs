# NOOS Deliberation Harness — Self-Bootstrap Runtime Working Model v0

> Status: Working Design Note / Not a frozen Review Candidate
>
> Purpose: capture the post-v4 design convergence derived from actual manual ChatGPT orchestration practice. This document intentionally does not modify or supersede frozen Review Candidate v4.

## 1. Core observation from current manual practice

The user is already manually acting as the missing Harness around ChatGPT Web.

The manual Harness work is primarily:

1. bind a conversation to a reasoning role and Work Item;
2. preserve the primary goal and scope;
3. issue lightweight continuation commands such as `Go`;
4. notice when the agent declares an external dependency such as Independent Review;
5. create/fork additional conversations for Review or Sedimentation work;
6. return external results to the correct Design Thread;
7. manage browser-level continuity: tabs, forks, conversation stabilization, renaming, refresh, and recovery;
8. keep durable design material in external documents near the Current authoritative document.

The human is **not normally choosing the Design Agent's next semantic design step**. The Design Agent generally proposes the next valuable direction itself. The human intervenes semantically only when the agent appears to drift from the main goal or crosses a true authority boundary.

This suggests a core product principle:

> NOOS should automate the user's coordination behavior, not replace the Design Agent's product reasoning.

---

## 2. Semantic state vs Harness state

### 2.1 Deliberation / Semantic State

Owned by the reasoning Agent and durable documents.

Examples:

- current proposed design;
- rationale;
- rejected alternatives;
- why a rejected alternative originally looked attractive;
- counterexamples;
- reopen conditions;
- unresolved questions;
- supporting evidence;
- working notes and decision memory.

NOOS should not independently infer or maintain competing structured truth for these concepts in v0.

### 2.2 Harness / Operational State

Owned by NOOS.

Examples:

- Work Item identity;
- primary goal and explicit scope constraints;
- conversation ↔ role ↔ Work Item binding;
- Current document reference;
- Companion / deliberation-memory document reference;
- browser tab/window identity;
- current provider conversation identity when stable;
- lifecycle operations such as fork, review dispatch, sedimentation dispatch, continuation, rollover;
- exact frozen target references where required;
- lightweight operation receipts.

Core rule:

> Agent decides meaning; Agent manages semantic documents; NOOS manages bindings, lifecycle, routing, and execution continuity.

---

## 3. External documents are the semantic durable state

The current direction prefers **Mode B**:

- Agent reads external documents through already-working connectors;
- Agent reasons about what should change;
- Agent writes the external documents itself;
- NOOS does not reimplement provider-specific document mutation.

Candidate and deliberation memory may therefore live directly in GitHub Markdown, GitBook, Notion, or another provider.

NOOS tracks references and operational identity rather than owning another canonical copy of semantic content.

### 3.1 Minimal physical document model

Semantic distinction:

1. **Current** — what we currently claim or propose;
2. **Working Memory** — material still worth considering;
3. **Decision Memory** — durable reasoning about important choices, especially rejected alternatives and pitfalls.

Physical v0 may use only two documents:

```text
Work Item
├── Current
└── Companion
    ├── Working Memory
    └── Decision Memory
```

The physical split is storage strategy, not semantic identity.

### 3.2 Negative knowledge is first-class memory

Rejected designs must not be treated as disposable closed items when they are likely to reappear.

A useful rejected-design record explains:

- what the rejected idea was;
- why it initially appeared attractive;
- what counterexample or tradeoff invalidated it;
- why the current direction is preferred;
- under what changed condition the rejected idea should be reconsidered.

This prevents future agents from repeatedly paying the same reasoning cost or falling into the same attractive trap.

---

## 4. Capture is not necessarily the primary memory operation

Earlier designs emphasized inline Capture proposals. Actual manual practice suggests a stronger operation:

## 4.1 Instant Capture

A low-frequency fallback:

> "This must not be lost right now."

May be manual or opportunistic. It is not required every turn and must not shape the natural Design response.

## 4.2 Sedimentation / Memory Flush

The primary durable-memory operation observed in real use:

```text
Main Design Conversation
    ↓ after many turns / context-risk boundary
Fork from current point
    ↓
Sedimentation Conversation
    ↓
read governance + existing durable docs
    ↓
retrospect over inherited deliberation context
    ↓
write missing high-value reasoning into Current-adjacent durable documents
    ↓
perform omission pass / deduplicate against already-recorded material
```

Important behavior:

- the fork inherits rich context from the Design Thread;
- the sedimentation reasoning does not pollute the future Design Thread context;
- only durable document changes need to persist across the branch boundary;
- the worker should preserve negative design knowledge and expensive rationale;
- it should avoid repeating material already present in durable documents;
- it should explicitly check for important omissions before finishing.

A key success criterion is:

> After Sedimentation, a future fresh Agent should depend materially less on the old conversation history.

---

## 5. Absorb as agent-driven semantic reconciliation

Absorb remains distinct from Sedimentation.

### Sediment goal

Reduce dependence on ephemeral conversation history by creating durable reasoning memory.

### Absorb goal

Bring the Current semantic head back into alignment with the latest reasoning and durable memory.

The agent may:

- update Current;
- compact or reorganize Companion;
- preserve durable rejected rationale;
- keep unresolved material rather than forcing a decision.

Working-document mutation is autonomous by default because it is reversible and non-promoted.

Human pre-acceptance is not the default for ordinary Absorb.

Human authority remains appropriate for higher-authority transitions such as Freeze / Promote / Publish.

General principle:

> Autonomy follows reversibility and authority.

---

## 6. `GO` is a continuation control token, not task planning

The Design Agent normally determines the next valuable semantic reasoning step itself.

NOOS should not tell it what design step to perform unless a human has explicitly changed scope.

`GO` means approximately:

> Continue the currently assigned reasoning work along the existing trajectory, under the same Primary Goal, Scope, and Role. Choose the next valuable reasoning step yourself. Do not silently expand the Work Item. If progress requires an external dependency or authority boundary, stop and declare it instead of inventing a new scope.

### 6.1 Session establishment vs in-session continuation

At conversation bootstrap / rollover, NOOS should provide:

- role;
- Work Item;
- Primary Goal;
- Scope / Non-goals;
- relevant durable document refs;
- authority boundaries.

During a healthy long-running Design Conversation, every `GO` should **not** repeatedly inject all documents and instructions. The current conversation trajectory is valuable context.

Durable refs are primarily bootstrap / recovery / on-demand context, not mandatory per-turn payload.

---

## 7. Goal anchoring without a semantic supervisor

The user currently performs occasional semantic drift correction manually. v0 should not add a second reasoning agent inside NOOS to judge every turn.

Instead, drift mitigation should begin with:

1. stable Primary Goal;
2. explicit Scope / Non-goals;
3. Agent-side continuation self-check;
4. periodic lightweight re-anchoring;
5. durable incorporation of explicit human scope corrections when appropriate.

### 7.1 Agent-side self-check

Before changing reasoning direction, the Design Agent should silently ask:

> Is this next step necessary to advance the current Primary Goal, or merely adjacent and interesting?

Normal aligned continuation should not emit administrative self-check text.

If the next step is adjacent but unnecessary, the Agent should stay on the current Work Item and may surface the adjacent idea as a possible later Work Item.

If progress requires Review, Human judgment, Evidence, or another external result, the Agent should stop autonomous continuation and declare the dependency.

### 7.2 Periodic re-anchor hypothesis

Do **not** inject a scope reminder every turn.

The current hypothesis is to use a sparse lifecycle hook / periodic re-anchor, for example after a configurable number of substantive Design turns or after context-changing events.

Candidate trigger points:

- every N substantive Design turns;
- after compaction;
- after rollover;
- after returning from Review;
- after a Human scope correction;
- before/after long forked worker operations when the main Design Thread resumes.

The exact turn interval should be treated as an experiment, not a semantic contract. A plausible starting range is roughly 6–12 substantive Design turns, with event-triggered re-anchors taking precedence over rigid counting.

The re-anchor should be short and should not ask the Agent to produce a visible alignment report.

---

## 8. Relationship to Claude Code-style hooks

The useful analogy is lifecycle interception, not copying Claude Code literally.

Claude Code exposes hooks at lifecycle boundaries such as session start, prompt submission, tool use, stop, and compaction. NOOS can adopt the same architectural idea:

> deterministic Harness logic runs at selected lifecycle boundaries, while semantic judgment remains with the primary Agent unless a specific evaluator is deliberately introduced later.

Potential NOOS lifecycle hook points:

```text
ConversationBound
BeforeContinue
AfterAgentTurn
BeforeFork
AfterForkStabilized
BeforeSediment
AfterSediment
BeforeReviewDispatch
AfterReviewReturn
BeforeCompact
AfterCompact
BeforeRollover
AfterRolloverBootstrap
```

v0 should keep the hook actions cheap and non-semantic where possible.

A periodic Goal Re-anchor can be implemented as one such hook without asking NOOS to evaluate the content of every turn.

---

## 9. Multi-tab browser execution model

The intended browser model should support multiple ChatGPT conversations concurrently.

Recommended conceptual shape:

```text
Chrome Extension
│
├── background/service worker
│   └── global coordination + tab/window lifecycle events
│
├── content script in ChatGPT Tab A
│   └── Design Thread observation/actuation
│
├── content script in ChatGPT Tab B
│   └── Reviewer Thread observation/actuation
│
├── content script in ChatGPT Tab C
│   └── Sedimentation Thread observation/actuation
│
└── ...
```

A tab does not need to be the currently active tab for a content script to exist and communicate with the extension.

NOOS should identify execution carriers with browser identities such as `tabId` / `windowId`, while provider conversation identity may stabilize later.

The browser UI may use either one window with many tabs or multiple windows. This is a UX choice rather than a fundamental extension limitation.

### 9.1 Important MV3 constraint

The extension background service worker is event-driven and may be terminated when idle. Therefore:

- do not treat service-worker memory as durable orchestration state;
- persist tab/conversation/role/Work Item bindings outside transient globals;
- design all browser events and messages to be restart-safe.

Chrome extension APIs support observing and manipulating tabs and injecting scripts across matching pages; this supports multiple concurrently managed ChatGPT tabs rather than only the currently active tab. The architecture should still tolerate tab discard/freeze/reload and service-worker restart. 

### 9.2 Recommended v0 UX

Prefer **one dedicated Chrome window containing multiple Harness-managed ChatGPT tabs** because it offers:

- low window-management overhead;
- easy visual grouping;
- natural parallel Design / Review / Sedimentation branches;
- extension-wide tab coordination.

Separate windows should remain supported, especially when the user wants visual separation, but should not be required by the architecture.

---

## 10. Browser Harness lifecycle: separate Carrier Runtime from Logical Control

Do not collapse browser execution state and reasoning-workflow state into one `tab_state`.

### 10.1 Carrier Runtime State

Purely observable / operational:

```text
ATTACHING
READY
GENERATING
STABILIZING
SUSPENDED
RECOVERING
BROKEN
CLOSED
```

Approximate semantics:

- `ATTACHING`: tab exists, but content script / provider-conversation identity is not yet established;
- `READY`: page is stable, input surface is usable, and generation is not running;
- `GENERATING`: ChatGPT is actively producing output;
- `STABILIZING`: generation, fork, navigation, or bootstrap just completed but URL / conversation identity / DOM still needs confirmation;
- `SUSPENDED`: tab has been frozen/discarded or otherwise cannot currently execute reliably;
- `RECOVERING`: Harness is refreshing/reactivating/rebinding the carrier;
- `BROKEN`: recovery failed or the page can no longer be controlled reliably;
- `CLOSED`: carrier no longer exists.

This layer should be derived from browser / DOM / extension events rather than semantic inference.

### 10.2 Logical Thread Control State

Workflow control, orthogonal to browser state:

```text
CONTINUE
WAIT_REVIEW
WAIT_HUMAN
WAIT_EVIDENCE
WAIT_WORKER
BOUNDARY_REACHED
```

This state should come from an explicit Agent/Human/Harness-declared boundary, not from DOM heuristics.

### 10.3 GO eligibility

The minimal rule is:

```text
Carrier == READY
AND
ThreadControl == CONTINUE
→ GO allowed
```

Examples:

| Carrier Runtime | Logical Control | Harness action |
|---|---|---|
| READY | CONTINUE | GO allowed |
| GENERATING | CONTINUE | wait |
| READY | WAIT_REVIEW | dispatch/wait Review |
| READY | WAIT_HUMAN | surface Human gate |
| SUSPENDED | any active control | recover carrier |
| BROKEN | any | report/rebind |
| READY | BOUNDARY_REACHED | stop |

Provider UI readiness does **not** imply semantic permission to continue.

---

## 11. Fork lifecycle

A newly opened tab is not yet a valid worker conversation.

Conceptual lifecycle:

```text
Parent READY
↓ Fork
Child tab created
↓
ATTACHING
↓ bootstrap interaction if required
provider conversation identity begins to exist
↓
STABILIZING
↓ resolve URL/title/conversation-list identity
↓ optional refresh/rebind validation
↓
READY
```

A fork should be considered stable only after the Harness can reliably re-identify the child conversation after navigation/reload/rebind, rather than merely because a tab was opened.

The exact detection method is provider-specific implementation detail and should not become product semantics.

---

## 12. Sparse Goal Re-anchor as a lifecycle hook

A periodic re-anchor is a safety mechanism, not a semantic supervisor.

Maintain a cheap operational counter such as:

```text
design_turns_since_anchor
```

A substantive Design turn may be defined operationally as one complete Design-Agent generation cycle initiated by normal continuation, not by semantic content scoring.

Candidate policy:

```text
re-anchor if:
    lifecycle_boundary_occurred
OR
    design_turns_since_anchor >= N
```

Event-triggered re-anchors should take precedence over rigid counting.

Potential lifecycle triggers include:

- review result returned;
- compaction;
- rollover;
- human scope correction;
- sedimentation return;
- long worker branch completion before resuming the main Design Thread.

`N` should remain experimental. A starting value around 8 substantive turns is plausible, within a broader 6–12 range.

The re-anchor itself should not request a visible alignment report.

---

## 13. Blocker Declaration Contract

The Harness only needs to know one semantic-control fact:

> Can this thread continue autonomously under its current Work Item, Role, Goal, and Scope?

The Harness does **not** need to know what exact semantic step the Agent intends to perform next.

### 13.1 Normal continuation is implicit

`CONTINUE` should be the default and need not appear in every Agent response.

Do not append mandatory control metadata such as:

```text
NEXT_STATE: CONTINUE
GOAL_ALIGNED: YES
```

to every normal answer.

That would create protocol noise and risk shaping the natural reasoning output.

### 13.2 Only exceptional boundaries need an explicit signal

Minimum candidate blocker vocabulary:

```text
NEEDS_REVIEW
NEEDS_HUMAN
NEEDS_EVIDENCE
NEEDS_EXTERNAL_WORKER
BOUNDARY_REACHED
```

These signals only answer why autonomous `GO` should stop. They must not encode the semantic next task.

Avoid expanding into categories such as `NEXT_ARCHITECTURE`, `NEXT_RESEARCH`, `NEXT_DOC_EDIT`, etc.; that would recreate semantic orchestration inside NOOS.

### 13.3 Natural explanation + machine control signal

When a boundary is reached, the Agent should still explain the reason naturally to the human.

Separately, the runtime should expose a tiny machine-readable signal when transport permits:

```text
NEEDS_REVIEW
```

or equivalent sidecar metadata.

The machine signal is not the explanation. NOOS should not parse the prose to recover meaning when a stable signal is available.

### 13.4 Transport independence

The semantic contract is:

> exceptional continuation boundaries can be signaled out-of-band.

The contract is **not**:

> a specific visible marker must appear in Markdown.

Transport options may evolve:

1. native tool-call / sidecar metadata if the runtime eventually exposes one;
2. extension-recognizable hidden or minimally rendered protocol block as an interim browser transport;
3. Human fallback when no reliable machine signal is available.

If a browser-encoded marker is used in v0, it is a transport workaround rather than long-term product semantics.

### 13.5 Fail-open vs fail-closed

A missing blocker signal is dangerous only in future autonomous Run Mode.

Therefore:

- **Step Mode v0:** absence of a blocker signal is acceptable because the Human still triggers the next `GO`;
- **Run Mode later:** repeated autonomous `GO` requires a stronger stop-signal contract and should fail conservatively when control-state confidence is insufficient.

This sequencing allows the Harness to validate browser lifecycle automation before relying on perfect semantic stop signaling.

---

## 14. Step Mode before autonomous Run Mode

### Step Mode v0

```text
Agent turn completes
↓
Carrier READY
↓
Human / explicit Harness GO
↓
next Agent turn
```

NOOS may automate:

- carrier observation;
- fork stabilization;
- tab/role bindings;
- periodic re-anchor;
- Review/Sedimentation worker creation;
- recovery.

But GO remains explicitly triggered.

### Later Run Mode

```text
READY + CONTINUE
→ GO
→ READY + CONTINUE
→ GO
→ ...
```

until:

```text
WAIT_REVIEW
WAIT_HUMAN
WAIT_EVIDENCE
WAIT_WORKER
BOUNDARY_REACHED
TURN_BUDGET
ERROR
```

Do not make autonomous Run Mode a prerequisite for the first self-bootstrapping slice.

---

## 15. Current proposed runtime rhythm

```text
Design Thread
    ↓
Agent reasons autonomously
    ↓
Agent says next useful direction
    ↓
If no external dependency:
    GO
    ↓
periodic/event-driven Goal re-anchor
    ↓
continue

If context / memory risk grows:
    SEDIMENT
    ↓
fork worker
    ↓
write durable memory
    ↓
main Design Thread remains clean

If Agent declares Independent Review needed:
    NEEDS_REVIEW
    ↓
spawn/fork Reviewer
    ↓
review exact target
    ↓
return Review result to Design Thread
    ↓
re-anchor
    ↓
GO / revise
```

NOOS does not need to understand the semantic content of each step. It manages the carriers and lifecycle around the agents.

---

## 16. What v0 should explicitly avoid

Do not require:

- NOOS semantic classification of every Design turn;
- NOOS deciding the Design Agent's next product-design step;
- per-turn goal alignment evaluation;
- a structured Capture item database as the source of semantic truth;
- duplicate semantic state in both NOOS and external documents;
- mandatory inline Capture markers;
- a heavy plan graph;
- a supervisor agent as a prerequisite for continuation;
- a provider-independent document mutation engine;
- mandatory per-turn control-signal tails;
- autonomous Run Mode before the stop-signal contract is validated.

---

## 17. Current open product questions

1. What exact DOM/browser observations are sufficient to infer `READY`, `GENERATING`, `STABILIZING`, and `BROKEN` robustly on ChatGPT Web?
2. What provider-specific evidence is sufficient to declare a forked conversation stable and re-identifiable?
3. What interim transport should carry exceptional blocker signals without polluting visible reasoning output?
4. How should the Human fallback work when a blocker signal cannot be machine-detected reliably?
5. Should Sedimentation always fork from the current Design Thread, or may NOOS sometimes reuse a dedicated long-lived Sedimentation Thread?
6. What minimal operation receipt should be recorded after Agent-driven external document writes?
7. At what point, after Step Mode is validated, is autonomous Run Mode justified?

---

## 18. Working north-star

> NOOS should replace the user's repetitive conversation coordination work while preserving ChatGPT as the primary semantic reasoning runtime.

The first implementation slice should be judged by whether it reduces manual tab/fork/continuation/review/sedimentation coordination without requiring NOOS itself to become another reasoning agent.
