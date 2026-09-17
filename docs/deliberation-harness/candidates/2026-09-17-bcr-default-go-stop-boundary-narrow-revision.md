# Bounded Continuation Run v0 — Default-Go × Stop-Boundary × 4-Turn Re-anchor Narrow Revision

> Status: `OWNER-DIRECTED DESIGN DELTA — NOT AUTHORITY`
>
> Date: 2026-09-17
>
> Relation: narrow revision to `futouyiba/noos-shuttle` `docs/deliberation-harness/bounded-continuation-run-v0-working-candidate.md` @ `1f03a8072dbf6e89724547265e59207955fda266`. This delta simplifies continuation semantics; it does not itself promote BCR to Authority.
>
> Human decision provenance: PR #17 comment `5716914285` — https://github.com/futouyiba/noos_docs/pull/17#issuecomment-5716914285. That comment is a durable GitHub transcription of the Human operator's 2026-09-17 ChatGPT Primary Design instruction; it is not a new review verdict.

## 1. Product semantic

```text
Go ×N
=
在当前对话已经形成的主线上，
最多替 Human 自动发送 N 次 continuation。

默认 continuation = "go"

不要求 Human 再输入 Goal。
不要求 Harness 生成下一步。
每 4 个 accepted continuation 做一次轻量防跑偏提示。
遇到真正的停止边界则停止。
```

The Harness owns boundedness, runtime safety, idempotency, stop-boundary detection, and provenance. The Working Assistant remains responsible for deciding how to continue the current reasoning mainline.

## 2. No Goal-entry requirement

Starting:

```text
Go
Go ×5
Go ×10
Go ×20
```

must not require the Human to re-enter:

```text
Goal
任务目标
本次 Run 要完成什么
下一步是什么
```

The local continuation intent is already supplied by the current Provider Conversation:

```text
continue the current Assistant's ongoing mainline
```

If a managed Logical Thread already has durable Goal / Scope / Authority Basis, those remain guardrails. Their existence is not a UI prerequisite for starting `Go ×N`.

Ordinary existing conversations without a formal NOOS Goal object may start bounded continuation directly.

Any prior design that derives a run-local Goal/Scope snapshot for a legacy chat must therefore not be treated as a mandatory setup step for `Go ×N`.

## 3. Mainline is not a new authority object

“Current mainline” does not introduce a new:

```text
Mainline object
Plan
NextAction
Task Tree
```

Nor does the Harness need to state:

```text
Assistant should do X next
```

Mainline here is only a continuity constraint:

> Continue the issue the Assistant is already advancing; do not proactively switch into optional side branches, a new goal, or additional scope.

It is not new Goal, Scope, Current, or semantic Authority.

## 4. Default continuation payload

Default payload remains literal:

```text
go
```

The Harness should not, every turn:

```text
summarize history
restate Goal
restate Scope
specify next action
generate a new task brief
```

Those behaviors would turn a continuation mechanism into a Planner.

## 5. Four-turn lightweight re-anchor

V0 uses:

```text
plain_go_reanchor_interval = 4
```

The counter advances on:

```text
accepted automated continuation user turns
```

not send attempts, Assistant responses, or retries.

Therefore:

```text
#1  go
#2  go
#3  go
#4  go + lightweight re-anchor

#5  go
#6  go
#7  go
#8  go + lightweight re-anchor
```

A continuation in `PREPARED`, `DISPATCHING`, `UNCERTAIN`, or `FAILED_SAFE` does not advance the accepted-continuation counter unless provider acceptance is sufficiently established under the normal SubmissionOperation contract.

## 6. Lightweight re-anchor semantics

Required semantic skeleton:

```text
go

[NOOS Re-anchor]
继续沿当前主线推进，不要偏离到可选支线。
如果当前主线已经完成，或下一步需要 Human 决策 / Review / 外部 Evidence，请停下。
[/NOOS Re-anchor]
```

Provider-specific wording may vary only if it preserves both required clauses:

```text
A. continue the current mainline; do not branch into optional scope
B. stop on completion or a Human / Review / Evidence boundary
```

The exact rendered re-anchor payload MUST be recorded verbatim in SubmissionOperation provenance so later review can distinguish the canonical intent from an adapter-specific rendering.

The re-anchor only:

```text
maintains current mainline
surfaces stop boundaries
```

It must not:

```text
invent next action
introduce new Closure obligation
change Goal
change Scope
choose between design alternatives
prioritize optional work
```

## 7. Stop-boundary detector: default continue, do not plan

V0 changes emphasis from:

```text
prove continuation is positively justified
```

to:

```text
continue by default unless a stop boundary is detected
```

This is a shift from positive planning authorization toward negative stop-boundary detection.

The detector's primary question is:

```text
Is there a clear reason not to send another "go"?
```

not:

```text
What exactly should the Assistant do next?
```

An explicit `next_action_hint` is not required for normal continuation eligibility. Lack of a formal next-step sentence is not, by itself, a stop condition.

The stop-boundary detector is a Harness semantic classifier operating on the completed stable Assistant turn plus current guardrail references when they exist. It is not a workflow role, Planner, or semantic authority source. Runtime-mechanical boundaries are taken from canonical runtime/operation facts, not inferred by the semantic classifier.

If a semantic boundary cannot be classified with sufficient confidence, the detector returns `SEMANTIC_CONTINUITY_UNCERTAIN`; automation fails closed and no next `go` is created.

## 8. Minimal continuation policy

Conceptually:

```text
AUTO_CONTINUE iff

ContinuationRun == ACTIVE
AND budget_remaining > 0
AND CurrentConversationBinding still matches Run binding
AND runtime permits another SubmissionOperation
AND previous continuation outcome is sufficiently reconciled
AND no Human intervention occurred
AND no stop boundary is detected
AND, if this Run starts on a post-rollover Provider Conversation,
    RESUME_ELIGIBLE is currently true at Run creation
```

The final predicate is a creation-time gate for a new post-rollover Run. `RESUME_ELIGIBLE` remains only a derived Continuity projection; it does not itself authorize `go`. The Human `Go ×N` action is the new continuation authorization.

The semantic detector does not need to prove that:

```text
a formal Goal object exists
a formal next action exists
one exact Closure successor exists
Harness knows how to solve the current problem
```

## 9. Stop boundaries and disposition

### 9.1 Mechanical boundaries

These are determined from canonical runtime / operation / binding facts:

```text
SUBMISSION_UNCERTAIN
RUNTIME_UNSAFE
CONVERSATION_REBASE_REQUIRED
BUDGET_EXHAUSTED
USER_INTERVENTION
```

Required disposition:

```text
SUBMISSION_UNCERTAIN      → HOLD current Run; reconcile; no next go
RUNTIME_UNSAFE            → HOLD current Run; same-conversation recovery may resume it
CONVERSATION_REBASE_REQUIRED
                          → END current Run; invoke Continuity boundary
BUDGET_EXHAUSTED          → END current Run
USER_INTERVENTION         → END/CANCEL current Run; remaining budget does not auto-resume
```

### 9.2 Semantic boundaries

The bounded semantic detector may emit:

```text
MAINLINE_COMPLETE
NEEDS_HUMAN
NEEDS_REVIEW
NEEDS_EVIDENCE
NEEDS_EXTERNAL
OPTIONAL_SCOPE_EXPANSION
CLEAR_SCOPE_DRIFT
STALLED_OR_REPEATING
SEMANTIC_CONTINUITY_UNCERTAIN
```

Minimum executable interpretation:

- `MAINLINE_COMPLETE`: the stable Assistant turn explicitly states the current requested mainline is complete, or completion is directly entailed by an already-defined completion criterion. Otherwise do not infer completion merely because the answer sounds conclusive.
- `NEEDS_HUMAN / NEEDS_REVIEW / NEEDS_EVIDENCE / NEEDS_EXTERNAL`: the stable Assistant turn explicitly identifies that dependency, or an existing durable gate requires it.
- `OPTIONAL_SCOPE_EXPANSION`: the Assistant proposes a separable optional branch that is not required to continue the current question.
- `CLEAR_SCOPE_DRIFT`: for managed threads, the Assistant proposes/starts work outside durable Scope; for ordinary chats, it abandons the currently advancing question for a distinct new topic. If that distinction is not clear, emit `SEMANTIC_CONTINUITY_UNCERTAIN` instead.
- `STALLED_OR_REPEATING`: after at least two consecutive accepted automated continuations, the stable Assistant turns substantially repeat the same unresolved content without adding a new decision, evidence item, discriminator, or narrowed uncertainty. This is a conservative dogfood heuristic, not semantic truth.
- `SEMANTIC_CONTINUITY_UNCERTAIN`: the detector cannot confidently distinguish safe mainline continuation from drift/completion/dependency.

Required disposition:

```text
MAINLINE_COMPLETE          → END current Run
OPTIONAL_SCOPE_EXPANSION   → END current Run; Human decides whether to authorize new scope
CLEAR_SCOPE_DRIFT          → END current Run; Human
NEEDS_*                    → HOLD/STOP automation; Human/review/evidence boundary
STALLED_OR_REPEATING       → HOLD/STOP automation; Human or explicit re-anchor decision
SEMANTIC_CONTINUITY_UNCERTAIN
                           → HOLD/STOP automation; Human
```

A semantic detector error must never create a new authority fact. False-positive stopping costs automation recall; false-negative continuation is bounded by budget, guardrails, every-four-turn re-anchor, and the next evaluation. Any `UNCERTAIN` case fails closed.

## 10. Assistant-declared next step

Typical case:

```text
Assistant:
“下一步我会检查 X，然后收敛 Y。”

Harness:
go
```

The Harness need not extract X into a `NextAction`, authorize X separately, or synthesize a new instruction telling the Assistant to perform X.

The Provider Conversation already carries the local reasoning momentum.

## 11. No explicit next-step sentence

If the Assistant completes one reasoning stage without explicitly stating a formal next step, but the current mainline is visibly not closed and no stop boundary is detected, another literal:

```text
go
```

remains eligible.

No `next_action_hint` is required.

## 12. Managed Goal / Scope role

Where formal Goal / Scope / Authority Basis already exist, they continue to constrain the Run.

Their role is:

```text
guardrail
```

not:

```text
Go ×N setup form
```

Existing Goal constrains continuation; Human need not restate Goal to start continuation.

## 13. Ordinary conversation role

For an existing ordinary conversation without a formal Goal object, current Provider Conversation context plus the current Assistant turn is sufficient to begin bounded continuation.

NOOS may still detect Human boundaries, obvious scope changes, runtime ambiguity, or continuity problems. Lack of formal Goal metadata alone does not require converting the conversation into a managed Work Item before `Go ×N` can run.

## 14. BCR × Conversation Continuity seam

Conversation rollover remains a hard boundary for the current Run:

```text
BCR on C1
→ CONVERSATION_REBASE_REQUIRED
→ current ContinuationRun ENDS
→ Continuity CONTINUITY_BOUNDARY
→ rollover workflow
→ C2
→ Resume Verification
→ derived RESUME_ELIGIBLE
→ STOP
```

Terminology mapping for this seam:

```text
BCR CONVERSATION_REBASE_REQUIRED
→ invokes Continuity CONTINUITY_BOUNDARY

Continuity CHECKPOINT_STALE / DESTINATION_CHANGED
→ rollover preparation remains blocked after the BCR Run has already ended
```

A new `Go ×N` on C2 is allowed only when:

```text
RESUME_ELIGIBLE == true
AND Human chooses Go ×N
```

Then the Human action creates a new bounded continuation authorization and a new Run.

If `RESUME_ELIGIBLE != true`, `Go ×N` must not create a Run. The system remains held on C2 and must resolve the Continuity failure/Human boundary first.

Even after successful resume, NOOS must **not** ask for Goal again. The successful Continuity Checkpoint + BOOTSTRAP + Resume Verification path supplies the recovered working position; the Human's `Go ×N` action supplies the new continuation authorization.

`RESUME_ELIGIBLE` itself remains non-authoritative and cannot dispatch a `go` without that new authorization and normal dispatch gates.

## 15. No cross-rollover budget carry

Old Run budget does not survive a binding change as actuation authority.

```text
old Run ENDED
→ Continuity reaches RESUME_ELIGIBLE
→ Human chooses new Go ×N
→ new authorization
→ new Run
```

This new Human authorization is a button/action, not a Goal-entry ceremony.

## 16. Product projection

Recommended UI:

```text
[ Go ]
[ Go ×5 ]
[ Go ×10 ]
[ Go ×20 ]
```

For a normal same-conversation start, selecting `Go ×N` should start directly.

For a post-rollover conversation, the controls may become actionable only when `RESUME_ELIGIBLE == true`; this is an eligibility gate, not a request to re-enter Goal.

Do not show a setup dialog asking:

```text
What is your goal?
Describe the task.
What should the Assistant do next?
```

Only request Human input when a real stop boundary requires Human information or judgment.

## 17. Dogfood acceptance

At minimum verify:

1. Existing ordinary conversation + no formal Goal object + `Go ×5` → Run starts without Goal prompt.
2. Assistant states an obvious next step → Harness sends literal `go`; no NextAction object required.
3. Assistant does not explicitly state next step but no stop boundary exists → continuation still eligible.
4. Accepted continuation #1/#2/#3 → literal `go`.
5. Accepted continuation #4 → `go` + lightweight mainline re-anchor; exact rendered payload is retained in provenance.
6. Failed/uncertain send does not incorrectly advance the four-turn accepted-continuation counter.
7. Re-anchor does not introduce a new task or optional scope.
8. Assistant requests Human decision → automation stops before next `go`.
9. Assistant explicitly declares current mainline complete → current Run ends.
10. Same-conversation runtime failure → Run holds while recovery occurs; no blind next `go`.
11. CurrentConversationBinding changes / `CONVERSATION_REBASE_REQUIRED` → Run ends; no next `go`.
12. After successful rollover and `RESUME_ELIGIBLE`, a new Human `Go ×N` starts without another Goal prompt.
13. After rollover with BOOTSTRAP failure, Hard Resume failure, Soft `MISMATCH`, or Soft `UNCERTAIN`, `Go ×N` cannot create a new Run.
14. `CLEAR_SCOPE_DRIFT` ambiguity becomes `SEMANTIC_CONTINUITY_UNCERTAIN`, not an invented confident classification.
15. Two substantially repeating accepted turns may trigger the conservative `STALLED_OR_REPEATING` hold; one repetitive turn alone does not.

## 18. Resulting V0 definition

```text
Go ×N
=
bounded Human-authorized repetition of "go"

using current conversation as local reasoning context,

with:
- runtime/idempotency safety,
- stop-boundary detection,
- every-4-accepted-turn lightweight re-anchor,
- fail-closed handling of real boundaries.

It is not:
- Goal re-entry,
- planning,
- next-action generation,
- autonomous task expansion.
```

## 19. Implementation direction

```text
READY_FOR_BOUNDED_VERTICAL_DOGFOOD
```

This is an owner-directed Candidate delta, not an Authority promotion. Repository integration still requires the normal independent review / merge workflow.