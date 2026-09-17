# NOOS Conversation Continuity — Health × Refresh × Checkpoint × Rollover

> Status: `PRIMARY_ADJUDICATED DESIGN CANDIDATE — NOT AUTHORITY`
>
> Date: 2026-09-17
>
> Implementation direction: `READY_FOR_BOUNDED_VERTICAL_DOGFOOD`
>
> This document records the Primary-adjudicated V0 design candidate for Conversation Continuity. It does **not** itself acquire Authority and does not modify existing Binding, Submission, BCR, Goal, Scope, Current, or governance contracts.
>
> Primary adjudication provenance: PR #17 comment `5716914285` — https://github.com/futouyiba/noos_docs/pull/17#issuecomment-5716914285. That comment is the durable GitHub transcription of the exact Human-delivered adjudication from the 2026-09-17 ChatGPT Primary Design conversation. The source chat was not itself a GitHub-addressable artifact; the comment records that provenance explicitly rather than pretending the adjudication originated on GitHub.

## 1. Primary adjudication

Delivery source:

```text
Who: Human operator / Primary Design authority for this task
Where originally delivered: ChatGPT Primary Design conversation, 2026-09-17
Durable GitHub transcription: PR #17 comment 5716914285
```

Exact decisive text:

```text
PRIMARY_ADJUDICATION

PD-1:
ACCEPT OPTION A
Support Human-assisted rollover in V0.

CORE CANDIDATE:
ACCEPT WITH NARROW REVISION

Required narrow revisions:
1. Clarify Checkpoint does not acquire Authority.
2. ResumeReceipt is BOOTSTRAP evidence/result, not new semantic authority object.
3. RESUME_ELIGIBLE is derived projection, not new authority object.
4. Human adoption accepts observed pre-adoption history; do not claim NOOS-proven safe establishment.

IMPLEMENTATION DIRECTION:
READY_FOR_BOUNDED_VERTICAL_DOGFOOD
after narrow revision.
```

This document incorporates all four required narrow changes. The durable transcription above is provenance for the Human adjudication; it is not an independent review approval and does not promote this Candidate into Authority.

## 2. Executive summary

Conversation Continuity is a bounded recovery workflow across existing NOOS contracts, not a new generic Conversation Health subsystem.

```text
Browser Carrier degradation
→ reattach
→ reopen same Provider Conversation
→ refresh only when refresh-safe

Semantic continuity risk
→ re-anchor same Provider Conversation
→ re-evaluate
→ Human if unresolved

Provider Conversation genuinely unusable
→ Continuity Checkpoint
→ Human-assisted new-conversation adoption
→ atomic CurrentConversationBinding rollover
→ BOOTSTRAP
→ Resume Verification
→ derived RESUME_ELIGIBLE
→ STOP
```

V0 introduces no ConversationHealth state machine, ContinuityAssessment object, generic Continuity Controller, automatic semantic supervisor, or cross-rollover continuation authority.

The main new persisted artifact is `ContinuityCheckpoint`. It is an immutable, content-addressed continuity transport artifact. It does not acquire Authority.

`ResumeReceipt` is BOOTSTRAP evidence/result. `RESUME_ELIGIBLE` is a derived eligibility projection. Neither grants independent semantic or actuation authority.

## 3. Current facts retained

### 3.1 Identity model

```text
Logical Thread
!= Provider Conversation
!= Browser Carrier
```

Reattach, reload, and reopen of the same Provider Conversation are not rollover.

Rollover means:

```text
same Logical Thread
+
different Provider Conversation
```

### 3.2 Current binding remains canonical ownership relation

The existing authoritative relation remains:

```text
CurrentConversationBinding
- logical_thread_id
- provider_conversation_ref
- binding_generation
```

Execution separately depends on the appropriate `ActuationLeaseAuthority` and successful dispatch claim. Conversation Continuity introduces no parallel binding authority.

### 3.3 Existing rollover/dispatch atomicity is reused

Execution-owning submission states such as `DISPATCHING`, `OBSERVED_ACCEPTED`, and `UNCERTAIN` block binding movement until reconciliation. Conversation Continuity does not weaken or duplicate this mechanism.

### 3.4 Browser recovery is distinct from rollover

Browser Carrier degradation does not itself justify Provider Conversation replacement. Runtime recovery should first try the same Provider Conversation.

### 3.5 ChatGPT establishment is not proven safe for full automation

Current evidence does not establish conforming `IDENTITY_FIRST` or technically guaranteed `TRANSPORT_ONLY_ESTABLISHMENT` for ordinary ChatGPT new-conversation creation. Prompt discipline is insufficient. Therefore V0 uses the adjudicated Human-assisted adoption fallback.

## 4. Failure domains, not a health score

V0 distinguishes:

```text
Browser / Carrier degradation
Provider Conversation degradation
Semantic continuity risk
```

No combined numeric health score is introduced.

### 4.1 Browser / Carrier degradation

Examples include content-script detachment, tab suspension, service-worker restart, DOM/runtime failure, page reload, and broken execution surface. Recovery stays on the same Provider Conversation whenever possible.

### 4.2 Provider Conversation degradation

Stronger conversation-specific evidence is required, such as known conversation explicitly unavailable, inability to reopen the known conversation on a healthy carrier, unrecoverable stable identity, or persistent conversation-specific access failure.

A generic provider outage, login problem, or latency spike does not prove the conversation itself is unusable.

### 4.3 Semantic continuity risk

Examples include forgotten constraints, stale Current, reopened rejected paths, scope drift, closure regression, or contradiction of the authoritative basis.

These are risk signals only. They do not prove provider context corruption and do not independently authorize rollover.

## 5. Recovery ladder

```text
WAIT
→ REATTACH
→ REOPEN_SAME
→ REFRESH_SAME_IF_SAFE
→ REANCHOR_SAME
→ CONTINUITY_BOUNDARY
→ ROLLOVER
→ HUMAN when required
```

The ladder is an escalation preference, not a requirement that every incident pass every stage.

- `WAIT`: transient or unresolved execution conditions.
- `REATTACH`: restore runtime observation/attachment only; no binding mutation.
- `REOPEN_SAME`: open the known current Provider Conversation on another Browser Carrier; binding generation unchanged.
- `REFRESH_SAME_IF_SAFE`: automatic refresh only with evidence that Human-local state such as an unsent draft or attachment will not be destroyed.
- `REANCHOR_SAME`: semantic continuity suspicion first uses the existing semantic maintenance path.
- `CONTINUITY_BOUNDARY`: stop new continuation actuation and prepare continuity transport; it does not itself switch binding.

## 6. ContinuityCheckpoint

### 6.1 Definition

```text
ContinuityCheckpoint
= immutable continuity transport artifact
```

Suggested content identity:

```text
checkpoint_fingerprint = hash(canonical checkpoint payload)
```

### 6.2 Explicit authority boundary

Normative invariant:

```text
ContinuityCheckpoint DOES NOT acquire Authority.
```

It may contain references to Authority, snapshots of references/fingerprints, advisory working-position material, and runtime provenance.

It may not become Goal, Scope, Current, supersede an Authority document, resolve Review/Human gates, grant continuation authority, grant rollover authority, or grant actuation authority.

If a Checkpoint conflicts with a currently authoritative source, current Authority wins and the Checkpoint becomes stale or invalid for resume.

### 6.3 Two-layer structure

#### Layer A — Authoritative Continuity Basis

Machine-projected references only:

```text
logical_thread_id
work_item_id
role

goal_ref
goal_revision/fingerprint

scope_ref
scope_revision/fingerprint

current_authority_refs[]
active_target_refs[]
pending_gate_refs[]

source_provider_conversation_ref
source_binding_generation
source_fence
```

The Checkpoint carries these references; it does not inherit their Authority.

#### Layer B — Working Position Digest

Advisory material:

```text
current focus
closure frontier snapshot
accepted decisions
rejected alternatives + reasons
open questions
recent reasoning pivots
```

Conflict rule:

```text
Authority source
>
Authoritative-reference projection in checkpoint
>
advisory Working Position Digest
```

Layer B is never its own ground truth. Every Layer-B claim that is correctness-relevant to resume SHOULD carry a basis reference to one or more of:

```text
current Authority / Current artifact
accepted Human / Review decision artifact
retrievable visible source turn
other durable evidence artifact
```

A digest claim that cannot be supported by such evidence remains advisory only. It MUST NOT be promoted by the verifier into an asserted recovered fact merely because the digest and ResumeReceipt agree with each other.

## 7. Source fence

Checkpoint creation freezes the observed tail of C1. Where practical, reuse existing submission/conversation baseline evidence rather than creating a second head-tracking subsystem.

Immediately before rollover, C1 is re-observed. If the relevant observable boundary changed:

```text
CHECKPOINT_STALE
```

The Checkpoint must be rebuilt or handed to Human review.

## 8. Human-assisted destination establishment

Primary Design accepts this V0 path:

```text
NOOS opens fresh carrier
→ Human performs establishment interaction
→ stable formal Provider Conversation identity appears
→ Human inspects the observed destination conversation/history
→ Human explicitly chooses Adopt
→ destination adoption fence captured
```

Critical interpretation:

```text
Human adoption
=
Human accepts the observed pre-adoption Provider Conversation history
as the destination carrier history that NOOS may adopt.
```

It does **not** mean NOOS proved establishment was safe, semantically neutral, quarantined, side-effect-free, or free of provider-side context mutation.

The pre-adoption history is outside the fully fenced NOOS semantic execution path. Human authorization accepts that observed history as part of the carrier being adopted.

## 9. Destination adoption fence

Human authorization is tied to the destination state the Human actually observed.

Conceptually:

```text
destination_adoption_fence =
provider_conversation_ref
+
observed conversation-head baseline
```

Immediately before binding commit, C2 is re-observed. If C2 changed after Human authorization:

```text
DESTINATION_CHANGED
→ do not commit rollover
→ require renewed Human acceptance
```

The destination fence is first-apply eligibility evidence, not Authority.

## 10. Final rollover gate

Before `C1@g7 → C2@g8`, establish:

```text
C1 source fence is still valid
C2 destination adoption fence is still valid
CurrentConversationBinding is still expected C1@g7
no execution-owning SubmissionOperation blocks movement
Human adoption authorization still corresponds to observed C2 boundary
```

Only then may existing binding mutation semantics apply.

## 11. Post-binding bootstrap

After binding commit, C2 becomes canonical current. A matching lease is established, then continuity payload is delivered through an ordinary controlled `SubmissionOperation(kind = BOOTSTRAP)`.

The bootstrap instruction asks the Agent to restore working position, report understanding, and not yet advance the substantive task.

If binding succeeds but bootstrap fails, C2 remains canonical current. C1 is not silently restored.

## 12. ResumeReceipt

`ResumeReceipt` is:

```text
BOOTSTRAP evidence/result
```

It is not Goal, Scope, Current, decision authority, continuation authority, binding authority, or actuation authority.

It records what the BOOTSTRAP execution returned so that the Harness can evaluate whether the new working conversation appears sufficiently aligned to permit continuation.

Conceptual payload:

```text
checkpoint_fingerprint
logical_thread_id
authority_refs_seen[]
current_focus restatement
pending_gate_ids[]
unresolved_questions[]
semantic_probe:
  core_open_problem
  key_constraints
  blocked_action
contradictions_or_unknowns[]
```

Its provenance remains connected to the concrete BOOTSTRAP `SubmissionOperation`.

## 13. Resume Verification

### 13.1 Hard verification

Deterministically compare checkpoint fingerprint, Logical Thread identity, Goal/Scope/Current references, pending gates, and required durable focus references. Any current Authority change may make the Checkpoint stale.

### 13.2 Soft verification

A bounded semantic verifier may only judge:

```text
NO_MISMATCH_DETECTED
MISMATCH
UNCERTAIN
```

Ground truth for Soft Verification is **not the Working Position Digest itself**. The verifier compares the BOOTSTRAP/ResumeReceipt restatement against the digest **and the digest's cited supporting evidence**: current Authority/Current artifacts, accepted Human/Review decision artifacts, retrievable visible source turns, and other durable evidence refs.

Required rule:

```text
required working-position claim
+ no retrievable supporting basis
→ UNCERTAIN
```

Agreement between two ungrounded summaries is not evidence of restored semantic continuity. Unsupported Layer-B material remains advisory and cannot, by itself, support `NO_MISMATCH_DETECTED` for a required claim.

The verifier may not rewrite Goal/Scope/Current, generate a new plan, resolve disputed design questions, invent missing decisions, or silently repair an unsupported digest.

## 14. RESUME_ELIGIBLE

`RESUME_ELIGIBLE` is explicitly a derived projection, not a new authority object.

Conceptually:

```text
RESUME_ELIGIBLE =
  current binding is C2@g8
  AND valid matching lease exists
  AND BOOTSTRAP operation completed
  AND runtime is READY
  AND Hard Resume Verification passed
  AND Soft Verification == NO_MISMATCH_DETECTED
  AND no unresolved conflicting execution exists
```

If an underlying fact changes, the projection may cease to hold.

Normative invariant:

```text
RESUME_ELIGIBLE
!= CONTINUE authority
!= GO authorization
!= Submission dispatch claim
```

The V0 continuity workflow ends at:

```text
RESUME_ELIGIBLE
→ STOP
```

## 15. End-to-end Human-assisted rollover

```text
C1 current
│
├─ attempt cheaper recovery
│   reattach / reopen / safe refresh / re-anchor
│
└─ recovery insufficient
    │
    ▼
CONTINUITY_BOUNDARY
    │
    ├─ settle unresolved old execution
    ├─ read canonical Goal / Scope / Current / gates
    ├─ capture source fence
    └─ build ContinuityCheckpoint
        │
        ▼
Human establishes C2
        │
        ▼
stable C2 identity appears
        │
        ▼
Human inspects observed pre-adoption history
and explicitly accepts/adopts it
        │
        ▼
capture destination adoption fence
        │
        ▼
revalidate source + destination boundaries
        │
        ▼
atomic CurrentConversationBinding commit
C1@g7 → C2@g8
        │
        ▼
assign matching lease
        │
        ▼
BOOTSTRAP SubmissionOperation
        │
        ▼
ResumeReceipt
        │
        ├─ Hard Verification
        └─ Soft Verification
        │
        ▼
derived RESUME_ELIGIBLE
        │
        ▼
STOP
```

## 16. Crash / race semantics

- C1 changes after checkpoint → source fence mismatch → `CHECKPOINT_STALE`.
- C2 changes after Human adoption → destination fence mismatch → `DESTINATION_CHANGED` → renewed Human acceptance required.
- Crash before binding commit → C1 remains current.
- Crash after binding commit but before BOOTSTRAP → C2 remains current; recover lease/bootstrap from C2.
- BOOTSTRAP uncertain → existing SubmissionOperation reconciliation; no blind resend.
- Stale old-C1 callback cannot reacquire authority.
- Genuine late semantic activity on superseded C1 → record divergence and require Human handling; do not auto-merge or reactivate C1.

## 17. BCR seam

Current BCR direction remains that a ContinuationRun pins Provider Conversation + binding generation. A conversation rollover ends that Run rather than carrying remaining budget across the new binding.

```text
BCR ACTIVE on C1
→ BCR emits CONVERSATION_REBASE_REQUIRED
→ current Run ENDED
→ Continuity enters CONTINUITY_BOUNDARY
→ rollover workflow
→ Resume Verification
→ derived RESUME_ELIGIBLE
→ STOP
```

Terminology mapping:

```text
BCR CONVERSATION_REBASE_REQUIRED
→ Continuity CONTINUITY_BOUNDARY

Continuity CHECKPOINT_STALE / DESTINATION_CHANGED
→ rollover remains blocked after the old BCR Run has already ended
```

A later `Go` or `Go×N` after rollover is eligible to create a new Run only when:

```text
RESUME_ELIGIBLE == true
AND Human provides new Go / Go×N authorization
```

If bootstrap or Resume Verification fails, C2 may remain canonical current but `RESUME_ELIGIBLE` is false; no new BCR Run may be created on that basis. This is a mechanical Continuity gate, not a semantic evaluator guess.

`RESUME_ELIGIBLE` still grants no continuation authority by itself; the Human action is the new authorization and normal binding/lease/dispatch gates still apply.

## 18. Automation boundary

V0 may automate runtime observation, reattachment, same-conversation rediscovery, checkpoint projection, fingerprinting, source/destination stale detection, hard resume verification, submission reconciliation, and stale-event rejection.

It may bounded-automate existing contract-safe operations such as reopen-same, same-conversation lease movement, and re-anchor.

Refresh remains conditional on refresh-safety evidence.

Current ChatGPT conversation creation/adoption remains Human-assisted.

Semantic mismatch, unsupported required Working Position claims, Authority changes, ambiguous submission outcomes, or old/new conversation divergence remain fail-closed Human boundaries.

## 19. Key invariants

1. Browser Carrier failure != Provider Conversation failure.
2. Long conversation != unhealthy conversation.
3. Semantic risk != proof of provider context failure.
4. Reattach/reopen/refresh same conversation != rollover.
5. Semantic risk prefers re-anchor before rollover.
6. Latency alone never authorizes rollover.
7. Unresolved execution ownership blocks binding movement.
8. ContinuityCheckpoint never acquires Authority.
9. A Checkpoint only transports/references Authority.
10. Working Position Digest is advisory.
11. Checkpoint fingerprint proves artifact identity, not semantic correctness or Authority.
12. Source fence change invalidates rollover preparation.
13. Destination change after Human acceptance invalidates that adoption authorization.
14. Human-assisted adoption accepts observed pre-adoption history; NOOS does not certify that establishment as safe/quarantined.
15. ResumeReceipt is BOOTSTRAP evidence/result.
16. ResumeReceipt does not acquire semantic Authority.
17. Resume Verification detects mismatch; it does not create correct semantic state.
18. Unsupported required Layer-B claims force `UNCERTAIN`; summary agreement cannot self-authorize continuity.
19. RESUME_ELIGIBLE is a derived projection.
20. RESUME_ELIGIBLE grants no independent actuation authority.
21. BOOTSTRAP success != automatic continuation.
22. Successful binding switch is not silently rolled back if later bootstrap/verification fails.
23. Superseded conversations cannot regain authority from stale callbacks.
24. Full transcript is not injected by default.
25. Private chain-of-thought is not continuity transport material.
26. BCR Run does not cross Provider Conversation rollover in V0.
27. A post-rollover BCR Run cannot be created until `RESUME_ELIGIBLE == true` and a new Human continuation authorization exists.

## 20. Primary decisions

```text
OPEN_PRIMARY_DECISIONS: NONE
```

PD-1 is adjudicated by the Human decision recorded at PR #17 comment `5716914285`:

```text
ACCEPT OPTION A
Support Human-assisted rollover in V0.
```

Future fully automated rollover remains evidence-dependent and is outside this adjudication.

## 21. Bounded vertical dogfood evidence

Verify at least:

1. Browser-only failure recovers without rollover.
2. Long-but-stable conversation does not trigger rollover.
3. C1 changes after checkpoint → `CHECKPOINT_STALE`.
4. C2 changes after Human adoption → `DESTINATION_CHANGED`.
5. Human adoption is recorded as acceptance of observed pre-adoption history without NOOS-safe-establishment claim.
6. Wrong Current / Goal / Scope reference → Hard Resume Verification failure.
7. Omitted or unsupported required semantic constraint → Soft `MISMATCH` or `UNCERTAIN`, never self-confirmation from the digest alone.
8. Crash after binding commit → C2 remains canonical current.
9. BOOTSTRAP `UNCERTAIN` → reconcile, no blind resend.
10. Late C1 callback → no authority mutation.
11. Duplicate C1/C2 browser carriers → only canonical binding + lease + dispatch claim may actuate.
12. `RESUME_ELIGIBLE` recomputes false when an underlying eligibility fact becomes false.
13. `RESUME_ELIGIBLE` alone cannot dispatch a GO.
14. After rollover with BOOTSTRAP failure or Resume Verification not passing, a new `Go ×N` cannot create a BCR Run.
15. After successful rollover + current `RESUME_ELIGIBLE`, Human `Go ×N` may create a new Run without Goal re-entry.

## 22. Recommended implementation slice

Implement only:

```text
ContinuityCheckpoint canonical schema
Authority-reference shell projection
advisory Working Position Digest with basis refs for required claims
content fingerprint
C1 source fence
Human-assisted C2 establishment/adoption record
explicit Human acceptance of observed pre-adoption history
C2 destination adoption fence
existing reducer binding switch
existing generation-scoped lease assignment
BOOTSTRAP SubmissionOperation
ResumeReceipt as BOOTSTRAP result evidence
Hard Resume Verification
Soft semantic mismatch detector against cited supporting evidence
derived RESUME_ELIGIBLE projection
post-rollover BCR creation gate on RESUME_ELIGIBLE + new Human authorization
STOP — no automatic continuation
```

Explicitly out of scope:

```text
Conversation Health score
new ContinuityAssessment
generic orchestrating Controller
automatic semantic-triggered rollover
unsafe auto-refresh
claiming Human establishment is NOOS-safe
automatic ChatGPT preactivation
cross-rollover BCR Run continuation
Hub-wide state migration
Goal/Scope mutation
automatic GO after Resume Verification
```

## 23. Repository/lifecycle boundary

The current file location under `docs/deliberation-harness/candidates/` is only the placement used by PR #17. This Candidate does not define a repository-wide candidate directory convention, promotion lifecycle, or Harness governance authority.

Repository integration remains subject to the canonical repo review/merge workflow. Harness-level promotion/closure is outside this Candidate and outside the scope of `docs/agent-workflow.md` unless separately defined by an appropriate authority.

## 24. Disposition

```text
PRIMARY ADJUDICATION PROVENANCE RECORDED @ PR#17 comment 5716914285
CORE CANDIDATE: ACCEPTED WITH REQUIRED NARROW REVISION
PD-1: OPTION A ACCEPTED
OPEN PRIMARY DECISIONS: NONE
NEXT: READY_FOR_BOUNDED_VERTICAL_DOGFOOD AFTER REVIEW/INTEGRATION GATES
```

This document remains a non-Authority Candidate. Its repository integration requires independent review of the exact revised head and the normal merge gate.