# NOOS Deliberation Harness — V1 Primary Design Sedimentation Candidate

> Status: Sedimentation Candidate / reasoning memory / not authoritative Contract
>
> Purpose: preserve high-value rationale, rejected alternatives, counterexamples, assumptions, reopen conditions, and phase-transition logic from the long-running Primary Design conversation that led to `READY_FOR_BOUNDED_V1_IMPLEMENTATION_EXPERIMENT`.
>
> This document should be curated against current authoritative contracts before promotion into durable Current / Companion / Decision Memory. It intentionally avoids restating full contract mechanics when the durable contract already carries the operational rule.

---

## 1. Current phase boundary

The V1 design phase reached a natural stop condition after:

```text
existing Harness concepts/contracts
→ first E2E walkthrough
→ input-boundary failure discovered
→ corrected E2E walkthrough
→ 5 genuine implementation-blocking design seams
→ binding/activation closure
→ binding narrow review and rework
→ binding review PASS
→ child-result delivery closure
→ delivery narrow review PASS
→ review-target provenance correction PASS
→ final E2E readiness confirmation
→ READY_FOR_BOUNDED_V1_IMPLEMENTATION_EXPERIMENT
```

Current working interpretation:

> Known implementation-blocking V1 Design GAP = 0.

This means all design gaps identified by the corrected E2E validation have explicit closure contracts and review evidence. It does **not** mean the system is proven free of unknown implementation problems.

The design mode should therefore change from:

```text
Design-led validation
```

to:

```text
Implementation-led learning
```

The default now is **not** to keep inventing abstractions. New design work should reopen only when implementation evidence demonstrates that a contract assumption is false, two contracts cannot be jointly implemented, an authority/semantic choice is unresolved, or a materially different product tradeoff must be made.

---

## 2. Why the E2E walkthrough mattered

Before the walkthrough, many important mechanisms already existed as separate contracts:

- Logical Thread / Provider Conversation / Browser Carrier separation;
- Submission idempotency;
- Goal Re-anchor;
- child worker lifecycle;
- Freeze and Review target identity;
- rollover and current-carrier routing;
- Human authority boundaries.

The remaining risk was compositional:

> individually plausible contracts do not imply that the complete workflow is safe when races, restart, rollover, duplicate tabs, and child return happen together.

The E2E walkthrough was therefore used as a system-integration thought experiment rather than as another feature-design exercise.

That method should be reused at future major phase boundaries: simulate one concrete end-to-end lifecycle and force every transition to name identity, authority, durable state, retry/recovery semantics, and implementation evidence.

---

## 3. Important dogfood failure: correct reasoning with incomplete durable context

The first walkthrough (`v0`) incorrectly reported seven large Design GAPs because the worker inspected `noos-shuttle` as if it were the full design authority and did not load the relevant contracts from `noos_docs`.

This failure was useful rather than merely procedural.

The worker behaved correctly under its observable inputs:

```text
contract unavailable to worker
→ worker refuses to invent semantics
→ worker reports GAP
```

The actual failure was context provisioning:

```text
capable Agent
+
incomplete/wrong durable refs
→ internally reasonable but globally wrong conclusion
```

This is a concrete dogfood example for the Harness itself.

### Durable lesson

> Agent intelligence cannot compensate reliably for missing authoritative context. NOOS must provision correct durable refs and authority boundaries; the Agent should reason from those inputs rather than infer hidden project state.

### Reopen / implementation relevance

When building worker bootstrap/context delivery, test not just whether the Agent receives *some* context, but whether it receives the correct authority-ranked set:

```text
Working Design Contracts
Implementation/runtime evidence
frozen authority artifacts
worker-specific scope
```

A fresh worker should not have to discover the project’s source-of-truth topology by guessing repository layout.

---

## 4. Core ownership split: semantic state vs Harness state

A central design simplification was to resist making NOOS a second semantic brain.

### Semantic / deliberation state

Owned by Agent reasoning plus durable external documents:

- current design meaning;
- rationale;
- rejected alternatives;
- counterexamples;
- reopen conditions;
- open questions;
- evidence interpretation;
- working and decision memory.

### Harness / operational state

Owned by NOOS:

- Work Item identity;
- Primary Goal and Scope/Non-goals references;
- Logical Thread identity;
- conversation/carrier bindings;
- binding/lease generations;
- SubmissionOperation lifecycle;
- child lifecycle and routing;
- frozen refs / receipts;
- execution and recovery continuity.

The intended separation is:

> **Agent manages semantic state; NOOS manages operational state.**

This avoids a competing semantic truth inside the Harness, which would require NOOS to continuously interpret whether a design decision, rejected alternative, or review issue is semantically current.

### Rejected alternative

A more ambitious model in which NOOS independently maintains semantic claims, conflict detection, and automatic multi-agent arbitration was deliberately deferred.

Why it looked attractive:

- apparently stronger automation;
- easier dashboards/querying;
- potential automatic conflict resolution.

Why V1 rejects it:

- duplicates semantic truth already maintained by Agents/documents;
- requires semantic interpretation in the control plane;
- creates authority ambiguity;
- substantially increases correctness burden before the browser/runtime loop is validated.

### Reopen condition

Revisit only if implementation repeatedly demonstrates that durable docs plus Agent reasoning cannot provide sufficient semantic continuity, or if autonomous multi-agent fan-in becomes a concrete product requirement rather than a speculative future capability.

---

## 5. Conversation is a carrier, not the durable thread identity

A major conceptual anchor is:

```text
Logical Thread
!= Provider Conversation
!= Browser Carrier
```

This was not just nomenclature. It is what makes rollover, duplicate-tab recovery, service-worker restart, and child-result routing coherent.

### Why this matters

If browser tab identity were the thread identity:

- reload/reattach would look like a new worker;
- duplicate tabs would create duplicate logical ownership;
- child results would route to stale tabs;
- rollover would destroy continuity.

If Provider Conversation identity were the thread identity:

- rollover would incorrectly create a new semantic workstream;
- long-lived reasoning would be coupled to provider context limits.

The durable thread therefore owns continuity while conversations and tabs are replaceable execution carriers.

### Reopen condition

Only revisit if a future provider exposes a durable provider-native agent/thread primitive with stronger semantics than current conversations and it becomes advantageous to map directly to that primitive.

---

## 6. Why CurrentConversationBinding became the canonical truth

The corrected E2E surfaced that several plausible “current” fields could contradict each other:

```text
LogicalThread.active_conversation_ref
ProviderConversation.logical_thread_id
conversation ACTIVE/SUPERSEDED
binding_epoch/generation
```

The simplification was to make one canonical relation authoritative and everything else derived/historical.

The deeper rationale is:

> identity ownership must have one commit point; multiple synchronized current-state fields create contradiction risk and make race recovery ambiguous.

This is why `CurrentConversationBinding` is not merely another record. It is the answer to the single question:

> Which Provider Conversation is the current continuation carrier for this Logical Thread?

Presentation statuses and reverse lookups should never become independent write authorities.

---

## 7. Binding is not execution readiness

Another important simplification was separating:

```text
current ownership
```

from:

```text
permission/readiness to execute
```

A conversation can already be the current bound carrier but still be unsafe to GO because:

- semantic bootstrap for the new binding generation is incomplete;
- no READY carrier is attached;
- the current carrier lacks the canonical actuation lease;
- LogicalControl is waiting;
- an unresolved SubmissionOperation exists.

Why this separation matters:

- avoids overloading `ACTIVE` with identity and runtime semantics;
- allows a committed rollover to remain correct even if bootstrap later fails;
- recovery can retry/reconcile bootstrap without reactivating the old conversation;
- makes readiness a conjunction of independently observable gates rather than a vague lifecycle flag.

### Rejected alternative

Treating `ACTIVE conversation` as both “current owner” and “ready to execute.”

Why rejected:

- creates ambiguous intermediate states;
- makes post-commit bootstrap failure difficult to represent;
- encourages stale carriers to regain authority based on UI readiness alone.

---

## 8. The rollover/dispatch race changed the safety model

An initially tempting rule was:

```text
check current binding
check lease
check READY
→ click Send
```

The narrow review correctly attacked this as a time-of-check/time-of-use race.

Example:

```text
1. Shuttle checks C1@g7 is current and tab A has the lease.
2. Rollover commits L1 -> C2@g8.
3. Before local code notices, tab A clicks Send into C1.
```

Everything was “correct at check time,” but execution authority became stale before use.

### Durable lesson

> Runtime observations are evidence, not authority. The right to *begin* provider actuation must itself be durably claimed inside the same atomic authority domain that controls binding and lease movement.

This led to the `claim_submission_dispatch` concept and the atomic competition among:

- dispatch claim;
- current-binding switch;
- actuation-lease transfer.

This should be remembered as a general NOOS pattern:

> **When a local adapter action can become unsafe between check and actuation, move the authority-to-begin-action into the durable reducer boundary rather than adding more local checks.**

---

## 9. Why `OBSERVED_ACCEPTED` still blocks rollover

A subtle design correction was that provider acceptance is not execution completion.

After the user message is visibly accepted:

- the Agent may still be generating;
- tools/connectors may still execute;
- external writes may still occur.

Therefore binding/lease movement must remain blocked for execution-owning states including:

```text
DISPATCHING
OBSERVED_ACCEPTED
UNCERTAIN
```

The deeper principle is:

> ownership transfer must wait until the old execution carrier no longer owns unresolved side-effect potential, not merely until transport acceptance is known.

---

## 10. Why pre-activation cannot rely on prompt discipline

A provisional new conversation can exist while the old conversation is still current. If normal semantic bootstrap runs before binding commit, both can become meaningfully side-effect-capable.

A tempting workaround is to send a prompt like:

> “Only initialize context; do not use tools or modify anything.”

This was rejected as a safety boundary.

Why:

- prompt compliance is semantic/model behavior, not an enforcement primitive;
- the normal Agent environment may still expose tools, connectors, delegated work, or authority-bearing refs;
- a failure would create two simultaneous semantic executors before ownership commits.

Conforming automated activation therefore requires either:

```text
IDENTITY_FIRST
```

or a genuinely technical:

```text
TRANSPORT_ONLY_ESTABLISHMENT
```

where capabilities/payload are actually fenced by adapter/policy mechanisms.

If the provider requires an ordinary executable Agent turn before stable conversation identity exists and NOOS cannot technically quarantine that turn, automated activation should fail closed and use Human-assisted adoption.

### Important product stance

> Do not make the design “work on paper” by pretending ChatGPT Web exposes primitives it does not expose.

Implementation Slice 0 should empirically test the actual conversation-identity establishment behavior and feed evidence back into this assumption.

---

## 11. Result Delivery: reuse transport protocol instead of creating a second system

The final known E2E seam concerned returning an immutable child result, especially Review Result, into the current parent conversation without duplicate insertion after restart/ack loss/rollover.

A plausible but rejected solution was a separate ResultDelivery state machine parallel to SubmissionOperation.

Why it looked attractive:

- delivery has its own lifecycle language;
- easy to model acknowledgements explicitly.

Why rejected:

- duplicates persist-before-actuate, dispatch fencing, reconciliation, and uncertainty semantics;
- creates two transport protocols that can diverge;
- increases implementation and recovery surface.

The chosen simplification:

```text
SubmissionOperation.kind = DELIVER_CHILD_RESULT
```

with only child-result-specific identity/receipt semantics layered on top.

General lesson:

> specialize existing execution primitives before inventing parallel protocols.

---

## 12. Why ResultDeliveryKey must not include binding generation

The stable identity of a logical return is:

```text
(result_id, destination_logical_thread_id)
```

The parent’s current conversation/binding generation is late-bound only for the concrete dispatch attempt.

This distinction prevents a subtle duplicate bug:

```text
R17 intended for L1
L1 current = C1@g7
rollover
L1 current = C3@g8
```

If generation were part of the dedup key, the system could treat:

```text
(R17, L1, g7)
(R17, L1, g8)
```

as two legitimate deliveries.

But rollover changes **where** an undelivered result should go; it must not change **what logical delivery this is**.

General lesson:

> keep semantic/logical operation identity independent from replaceable execution-carrier identity.

---

## 13. Transport completion is not semantic acceptance

Several layers must remain separate:

```text
child RESULT_READY
→ result inserted into parent conversation
→ result-bearing parent Agent turn completes
→ parent reasons about result
→ semantic acceptance/revision decision
```

`INSERTED` means transport evidence proves the result entered the recipient conversation.

`COMPLETED` means the resulting parent Agent turn reached its runtime completion boundary.

Neither means:

- Review findings accepted;
- Candidate revised;
- promotion authorized;
- `LogicalControl = CONTINUE` automatically.

The deeper authority rule is:

> transport may deliver evidence; Harness may clear a mechanical wait; semantic judgment remains with the Primary Design Agent/Human authority boundary.

This is another instance of the broader separation between operational orchestration and semantic authority.

---

## 14. Why V1 begins with Step Mode

Step Mode is deliberately conservative:

```text
Human triggers GO
→ durable operation
→ one dispatch
→ observe/reconcile
→ stop at READY
```

Why not immediately autonomous Run Mode:

- provider browser automation has imperfect observability;
- blocker signaling is not yet empirically validated;
- uncertain delivery is safer to pause than to over-automate;
- Step Mode gives immediate dogfood value while keeping failures visible and recoverable.

The design criterion was not “maximize autonomy early.” It was:

> maximize useful learning while bounding the consequence of uncertain browser/provider behavior.

### Reopen condition

Run Mode becomes reasonable only after implementation evidence demonstrates reliable observation, dispatch reconciliation, blocker handling, and recovery across multiple real sessions.

---

## 15. Why Sedimentation and Independent Review use different child creation modes

The manual workflow revealed two different context needs.

### Sedimentation = FORKED

Sedimentation benefits from inherited conversational history because its job is to recover high-value reasoning that may not yet be durable.

### Independent Reviewer = FRESH

Review independence is weakened if the Reviewer inherits the full Designer reasoning trajectory. It should receive explicit durable inputs: frozen target, goal/scope, reviewer contract, selected evidence.

The important design principle is not merely “fork vs fresh.” It is:

> context inheritance should follow the epistemic purpose of the worker.

A worker needing latent history may fork; a worker whose value depends on independence should bootstrap fresh from durable authority-ranked inputs.

---

## 16. Why the Primary Design Thread should become scarce after V1 readiness

During invention, a long conversational trajectory is valuable because new concepts are still being compared and reframed.

After readiness, using the Primary Design conversation for routine implementation bookkeeping becomes harmful:

- context fills with logs and narrow patches;
- design rationale becomes harder to retrieve;
- every implementation detail tempts unnecessary architecture changes;
- the primary reasoning thread stops being a high-value semantic authority surface.

The preferred operating model now is:

```text
Primary Design Thread
→ only true design decisions / authority / cross-contract tradeoffs

GPT Work workers
→ narrow review, synthesis, evidence analysis, document curation, implementation briefs

Codex / implementation agents
→ bounded implementation slices and tests
```

This is intentionally close to the Harness model being built.

---

## 17. Triggers for returning to Primary Design deliberation

Routine implementation problems should not automatically reopen design.

Return to Primary Design only when at least one of the following occurs.

### A. Provider reality falsifies a contract assumption

Example:

- stable conversation identity cannot be obtained in the assumed activation sequence;
- READY/GENERATING observation is materially less deterministic than expected.

### B. Two contracts are individually sensible but jointly unimplementable

Example:

- binding safety and submission recovery demand mutually incompatible ordering in the real browser/runtime.

### C. A new semantic/authority choice appears

Example:

- should a returned result merely clear a mechanical dependency or also trigger a semantic transition?

### D. Multiple materially different product tradeoffs remain

Example:

- stronger automatic recovery vs simpler/manual recovery has significant UX or safety consequences.

### E. A bounded worker explicitly reports `NEEDS_DESIGN_DECISION`

The worker should return:

- observed facts;
- reproduction/counterexample;
- affected contracts;
- candidate options;
- why it cannot safely choose within delegated scope.

Primary Design then resolves only that seam.

---

## 18. Implementation-led learning discipline

After `READY_FOR_BOUNDED_V1_IMPLEMENTATION_EXPERIMENT`, the loop should become:

```text
contract
→ bounded implementation slice
→ real provider/runtime evidence
→ conformance check
   ├─ matches assumptions → next slice
   └─ concrete mismatch → narrow design reopen
```

Avoid the opposite anti-pattern:

```text
READY
→ continue imagining more edge cases indefinitely
→ add more contracts
→ delay contact with real provider behavior
```

The current contracts are not a claim of final product completeness. They are sufficient scaffolding for bounded experiments.

Implementation should now become the primary generator of new information.

---

## 19. Recommended implementation slicing rationale

The initial sequence should remain intentionally narrow.

### Slice 0 — Browser observation / carrier-binding experiment

Purpose:

> validate the lowest-level empirical assumptions before building higher-level durable orchestration.

Questions include:

- can provider conversation identity be observed stably?
- what survives reload?
- how do duplicate tabs present?
- how repeatable are READY / GENERATING / STABILIZING transitions?
- when does a new/forked conversation obtain stable identity?

Do not treat Slice 0 as “implement the Harness.” It is an observation experiment.

### Slice 1 — Human-triggered GO + durable submission/reconciliation

Only after Slice 0 provides credible observation primitives:

```text
Human GO
→ durable SubmissionOperation
→ atomic dispatch claim
→ browser actuation
→ acceptance observation
→ UNCERTAIN reconciliation
```

This is the first small loop with meaningful dogfood value.

### Later slices

- FORKED Sedimentation worker;
- FRESH Reviewer worker;
- automated rollover;
- child/result return path.

These should not block Slice 0/1 merely because the complete V1 runtime is not yet implemented.

---

## 20. Important rejected process alternative: “finish all design before any implementation”

The E2E process was necessary to close known composition seams, but there is a point at which more design ceases to reduce the dominant uncertainty.

At current readiness, the dominant uncertainty has shifted from conceptual correctness to provider/runtime reality.

Therefore the process decision is:

> do not require complete implementation-safe proof for every later worker flow before starting bounded lower slices.

This avoids using architecture work to postpone empirical learning.

---

## 21. Important process alternative: let GPT Work become another Primary Designer

GPT Work has proven valuable for:

- structured E2E walkthroughs;
- narrow adversarial review;
- evidence reconciliation;
- exact-target verification;
- documentation curation.

But it should not become an unconstrained parallel design center.

The first walkthrough demonstrated why: workers are only as good as their supplied durable context and authority boundary.

Preferred worker envelope:

```text
Role
Goal
Scope
Exact inputs
Authority
Expected output
Stop condition
```

Avoid prompts equivalent to:

> “Read the project and decide what the Harness should become next.”

unless deliberately opening a new design exploration.

---

## 22. Review provenance lesson

The final Result Delivery review contained a substantive PASS but initially referenced an exact commit in which the target file did not yet exist.

This did not invalidate the reasoning itself, but it invalidated the formal evidence binding until corrected.

Durable lesson:

> Review quality and Review Target identity are separate correctness dimensions.

A correct judgment bound to the wrong artifact revision is not sufficient governance evidence.

Future Review packets/results should preserve:

- exact path;
- exact commit/revision;
- target content/blob identity when practical;
- supporting dependency snapshot;
- scope of the verdict.

This is especially important when a later promotion or implementation decision cites “review PASS” as evidence.

---

## 23. Assumptions that implementation should actively test

These should be treated as empirical hypotheses rather than quietly forgotten design premises.

### Provider identity

- a stable Provider Conversation identity can be resolved/reconstructed reliably enough for binding;
- reload and duplicate tabs can be distinguished from semantic thread replacement.

### Runtime observation

- READY / GENERATING / STABILIZING can be derived conservatively from multiple provider signals;
- conversation/route changes can be detected before stale local state causes unsafe actuation.

### Reconciliation

- observable message-count/head/fingerprint evidence is sufficient for V1 conservative reconciliation in typical GO/result-delivery cases;
- ambiguous cases are rare enough that fail-closed/manual recovery remains usable.

### Preactivation

- at least one safe establishment/adoption workflow is practical for the provider, or Human-assisted adoption is acceptable for the affected early slice.

### Step Mode usability

- explicit Human GO does not create enough friction to invalidate the first dogfood loop;
- the safety/observability benefit outweighs lower autonomy during V1 learning.

If these assumptions fail materially, return evidence to Primary Design rather than hiding the mismatch in adapter heuristics.

---

## 24. What should *not* be reopened without evidence

The following should be considered settled V1 direction unless concrete implementation evidence contradicts them:

- one Primary Design Logical Thread per Work Item;
- bounded supporting child workers, not peer Design authority;
- Logical Thread distinct from Provider Conversation and Browser Carrier;
- semantic state primarily in Agent-managed durable documents, not duplicated inside Harness;
- current conversation binding has one canonical truth;
- binding is distinct from execution readiness;
- transport actuation authority is reducer/fence based, not local-UI-check based;
- prompt-only preactivation quarantine is non-conforming;
- Step Mode precedes autonomous Run Mode;
- Sedimentation primarily FORKED, Independent Review FRESH;
- child-result transport reuses SubmissionOperation;
- transport completion does not imply semantic acceptance;
- Review/frozen-target authority boundaries remain Human-controlled where specified.

Do not reopen these because an implementation agent prefers a simpler local shortcut. A reopen requires evidence that the current rule is infeasible, internally contradictory, or produces unacceptable product cost.

---

## 25. Suggested curation targets for the next Sedimentation Worker

This candidate intentionally mixes several memory classes. A follow-up curator should split/deduplicate rather than promote this document wholesale.

Likely durable destinations:

### Current / phase status

Keep concise:

- V1 final readiness status;
- implementation-led learning phase;
- Slice 0/1 next actions;
- known Design GAP count and meaning.

### Decision Memory / Companion

Preserve rationale such as:

- semantic vs operational ownership split;
- conversation-as-carrier model;
- binding != execution readiness;
- TOCTOU → reducer dispatch claim lesson;
- prompt quarantine rejection;
- ResultDeliveryKey generation independence;
- transport vs semantic acceptance distinction;
- Step Mode rationale;
- FORKED vs FRESH epistemic rationale.

### Rejected alternatives

Preserve with reopen conditions:

- NOOS semantic supervisor / duplicate semantic truth;
- tab/conversation as durable logical identity;
- `ACTIVE` overloaded with ownership + readiness;
- check-before-click lease fencing;
- prompt-only preactivation safety;
- second parallel ResultDelivery protocol;
- binding-generation-scoped delivery identity;
- design-everything-before-implementation;
- unconstrained GPT Work as parallel Primary Designer.

### Implementation assumptions / reopen triggers

Create a compact checklist for Slice evidence review.

---

## 26. Sedimentation success criterion

This sedimentation is successful if a future fresh Primary Design Agent can understand, without reading the old chat transcript:

1. why the V1 architecture has its current boundaries;
2. which attractive alternatives were rejected and why;
3. which counterexamples forced the final safety rules;
4. what “Known Design GAP = 0” actually means;
5. why design should now stop expanding by default;
6. what concrete evidence is sufficient to reopen design;
7. how implementation workers, GPT Work reviewers, and Primary Design should divide responsibility.

The goal is not archival completeness. It is to preserve the reasoning needed to avoid repeating already-resolved design debates while still making future reopening principled.
