# NOOS Deliberation Harness — Review Candidate v2

> Status: **Review Candidate / Narrow Revision**
>
> GitBook review-target revision: `xixA3Ug2ZRPGVt1ejpYl`
>
> This revision narrows v1 based on independent product/workflow audit. It does not expand implementation scope.

## 1. v0 Product Goal

NOOS Deliberation Harness turns complex requirement discussion in ChatGPT Web into a reviewed specification that can be published to GitHub Markdown / Notion and safely handed to an implementation agent.

The goal is not to add process. It is to reduce deliberation coordination tax while protecting natural reasoning quality.

v0 should expose only four core user actions:

1. **Capture** — preserve useful material into the current Work Item Inbox.
2. **Absorb** — ask the reasoning worker to propose a coherent Working Candidate revision from selected material.
3. **Freeze & Review** — freeze an immutable Candidate revision and send it to an independent Reviewer.
4. **Promote** — mark the spec implementation-safe when readiness criteria are satisfied.

## 2. Harness Engagement and Active Work Item

### 2.1 Explicit Deliberation Scope

v0 does not use always-on governance.

The Harness enters managed deliberation mode only after the user explicitly activates an **Active Work Item**.

Minimum semantics:

- One Provider Conversation has at most one Active Work Item at a time.
- Capture, Working Candidate and Review belong to the current Active Work Item.
- The user can explicitly switch Active Work Item.
- Switching does not automatically migrate historical Captures, Reviews or Candidate state.
- One Work Item has one mutable Working Candidate Head by default.
- One Conversation may serve multiple Work Items sequentially, but never multiple Active Work Items simultaneously.

AI-inferred Work Item suggestions are a later optimization, not a v0 authority source.

### 2.2 Ordinary chat remains ordinary chat

Without an Active Work Item:

- Harness does not require Capture behavior.
- No Staging / Review workflow is created.
- Normal ChatGPT reasoning should not change because NOOS exists.

## 3. Working Candidate

Working Candidate is the current Work Item's single mutable semantic head:

> If we had to state what we currently believe now, the Working Candidate should independently express it.

It is not conversation history and not a Capture list. It has no Committed authority.

It may contain current proposal, rationale, assumptions, alternatives, open questions, acceptance scenarios and supporting references.

Freeze for Review produces an immutable Candidate Revision. Later Working Candidate changes do not mutate that Review Target.

## 4. Capture v0

### 4.1 Semantics

Capture only means:

> Preserve content that the user or reasoning worker believes may be useful later as pending material of the Active Work Item.

v0 does **not** require global semantic novelty detection. Duplicate Captures are allowed; Absorb may merge, deduplicate or discard them later.

### 4.2 Two entry paths

v0 must support:

- **Manual Capture** — user explicitly captures selected text or part of an answer.
- **Optional AI Capture Proposal** — ChatGPT may suggest content worth capturing, but this is an optional experiment.

The workflow must remain fully usable without AI markers.

### 4.3 Capture must be non-blocking

Core interaction invariant:

> **Capture Proposal must never block the main Conversation.**

The user can continue chatting without first processing proposals.

An unconfirmed AI proposal:

- does not modify Working Candidate;
- does not block the next turn;
- may be ignored, processed later, or discarded.

### 4.4 Marker is an experiment default, not contract

`◈`, Markdown block anchoring and per-turn budgets are no longer normative product semantics.

v0 only freezes these experience requirements:

- suggestions should be sparse;
- the suggested content must be visibly anchorable and editable;
- no duplicated capture summary is required at answer end;
- Capture presentation must not materially reshape the natural answer;
- marker/parsing failure silently degrades without disrupting discussion.

Exact marker syntax, parsing boundary and quantity budget are UX / implementation experiments.

## 5. Work Item Inbox

v0 uses one **Work Item Inbox** as the durable staging surface.

It does not create Section-local / Document-local / Unbound ownership layers.

Minimum item data:

- content;
- source reference;
- status;
- optional target hint.

Section is only an optional routing hint / view, not an ownership boundary.

Final document placement is decided during Absorb.

v0 does not implement a full Artifact-promotion workflow. Complex material may remain an external document, attachment or reference.

## 6. Absorb Transaction

Absorb is not append. It means:

> The reasoning worker uses Current Working Candidate + selected Inbox items to propose a coherent Candidate revision diff.

### 6.1 Minimum flow

1. User selects one or more pending Inbox items.
2. Reasoning worker produces a **Proposed Candidate Diff**.
3. User can Accept / Reject / Continue discussing.
4. Only after Accept:
   - Working Candidate updates atomically;
   - actually incorporated Inbox items become `absorbed`;
   - selected but unused items may stay `pending`, or become explicitly `deferred / rejected`.
5. Failed, cancelled or rejected proposals do not mutate Working Candidate or the original Inbox state.

### 6.2 Minimum item lifecycle

```text
pending
  ↓ selected for absorb
proposed
  ├─ accepted → absorbed
  ├─ rejected → pending | rejected
  └─ deferred → deferred
```

Absorb is the core transaction boundary of v0.

## 7. Freeze & Independent Review

Freeze for Review is a **version action**, not a formal governance gate.

The user may freeze a Candidate even with remaining Inbox items or Open Questions, provided the unresolved state is visible to the Reviewer.

Freeze produces:

- immutable Candidate Revision;
- Review Snapshot / relevant baseline;
- independent Reviewer assignment.

Reviewer reviews the frozen revision, never the mutable Working Head.

## 8. Review Return: Review Report + selected Review Notes

v0 does not normalize the full Review prose into an Issue tree.

### 8.1 Immutable Review Report

Reviewer output is preserved as an immutable Review Report for provenance and later reference.

### 8.2 Review Notes

Designer marks only findings that actually require continued work as **Review Notes**.

A Review Note may contain:

- source Review Report reference;
- quoted or summarized finding;
- optional severity;
- disposition: `address / dismiss-with-reason / defer`;
- optional target hint.

There is no separate durable Review Staging object. “Open Review Notes” is simply a Work Item/status view.

Core authority rule:

> Reviewer raises challenge; Designer reasons about response.

Review Notes never directly mutate Working Candidate.

v0 does not implement full issue lineage or re-review resolution state machines.

## 9. Open Questions: Lazy Classification

During normal discussion, an Open Question may remain simply an Open Question.

At Freeze / Readiness boundaries, the reasoning worker proposes batch classification.

v0 uses four classes:

1. **Decide now** — product-side decision required before safe implementation.
2. **Leave to implementation** — multiple implementations do not change important observable product behavior.
3. **Defer with trigger** — not decided in current version; must state the condition that reopens it.
4. **Needs human / evidence** — requires authority preference, experiment, research or external evidence.

`Defer with trigger` without a trigger is not considered fully classified.

## 10. One Readiness Checklist, Two Labels

v0 does not create separate Review-ready and Implementation-ready workflow gates.

One Readiness Assessment yields two labels.

### `reviewable`

Candidate is clear enough to be worth freezing and attacking.

This is advisory, not a hard prerequisite to Freeze.

### `implementation-safe`

An implementer who did not participate in prior deliberation can begin without having to make new important product decisions.

Only `Decide now / Product Blocking` items hard-block `implementation-safe`.

Checklist:

- Problem / user outcome is explicit.
- Core observable behavior is explicit.
- Semantic / authority boundaries are explicit.
- Failure / boundary behavior is sufficient to avoid product-semantic guessing.
- Remaining unknowns are lazily classified.
- Acceptance scenarios can validate core function and experience.

## 11. Cold-start Handoff Test

This is the key Implementation-ready acceptance test.

Give the Spec **alone** to an implementer / Coding Agent that did not participate in the deliberation and ask only:

1. Which remaining questions must the product side answer before implementation is safe?
2. Which questions have two or more technically reasonable implementations that produce materially different observable product behavior?

Any such question becomes `Decide now / Product Blocking`, so the Candidate is not `implementation-safe`.

This test evaluates the Spec only; v0 still does not manage Codex runtime handoff.

## 12. Decision Boundary

Final Spec separates:

### Decided here

Decisions that change user experience, product semantics, authority, state consistency or observable failure behavior.

### Implementation discretion

Choices where different implementations do not materially change important observable product behavior.

### Deferred with trigger

Product questions intentionally left outside the current version with an explicit reopen condition.

Practical test:

> If Codex could reasonably implement two different versions that are both technically valid but materially different as products, the decision is still a product question.

## 13. Acceptance Scenarios

### Functional

1. User can Manual Capture after activating a Work Item.
2. Capture enters that Work Item Inbox without modifying Working Candidate.
3. Absorb updates Candidate and item status only after the user accepts the Proposed Diff.
4. Frozen Review Target is not changed by later Working Candidate edits.
5. Review Report cannot directly mutate the document.
6. User can select a small number of findings as Review Notes for continued work.
7. A Product Blocking finding from cold-start test prevents `implementation-safe`.

### Experience

1. Without Active Work Item, ordinary ChatGPT discussion is unaffected by Harness workflow.
2. Missing, wrong or unparsable AI Capture Proposal does not disrupt discussion.
3. User can send the next message without processing a Capture Proposal.
4. During a real 30–60 minute requirement discussion, the user does not repeatedly maintain section routing, issue trees or multiple readiness gates.
5. Harness should not induce materially more duplicate summaries, protocol tails or capture-friendly rewriting from ChatGPT.

## 14. v0 Out of Scope

- always-on / AI-inferred Harness engagement;
- normative `◈` marker protocol;
- section-owned Staging;
- Artifact promotion workflow;
- durable Review Staging object;
- full issue lineage / re-review state machine;
- complex Multi-Agent Integration UI;
- per-turn Open Question governance;
- automatic `/go` orchestration;
- Continuation / Rollover;
- Sidecar tools;
- Codex session / runtime handoff.

## 15. v0 Irreducible Core

```text
Conversation
→ Explicit Active Work Item
→ Manual Capture / optional non-blocking Capture Proposal
→ Work Item Inbox
→ Absorb Proposed Diff
→ Human Accept into Working Candidate
→ Freeze immutable Candidate Revision
→ Independent Review Report
→ Selected Review Notes
→ Readiness Assessment + Cold-start Handoff Test
→ GitHub Markdown / Notion Spec
```

The goal of v0 is not to cover the complete Harness. It is to make this smallest loop light enough that it can immediately accelerate the next NOOS design cycle.