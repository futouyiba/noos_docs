# Bounded Continuation Run v0 — Default-Go × Stop-Boundary × 4-Turn Re-anchor Narrow Revision

> Status: `OWNER-DIRECTED DESIGN DELTA — NOT AUTHORITY`
>
> Date: 2026-09-17
>
> Relation: narrow revision to the current BCR Working Design Candidate. This delta simplifies continuation semantics; it does not itself promote BCR to Authority.

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

Canonical intent:

```text
go

[NOOS Re-anchor]
继续沿当前主线推进，不要偏离到可选支线。
如果当前主线已经完成，或下一步需要 Human 决策 / Review / 外部 Evidence，请停下。
[/NOOS Re-anchor]
```

Exact wording may adapt to the Provider/conversation, but the semantic payload remains narrow.

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

## 7. Evaluator philosophy: default continue, detect stop boundaries

V0 changes emphasis from:

```text
prove continuation is positively justified
```

to:

```text
continue by default unless a stop boundary is detected
```

This is a shift from positive planning authorization toward negative stop-boundary detection.

The evaluator's primary question is:

```text
Is there a clear reason not to send another "go"?
```

not:

```text
What exactly should the Assistant do next?
```

An explicit `next_action_hint` is not required for normal continuation eligibility. Lack of a formal next-step sentence is not, by itself, a stop condition.

The evaluator remains a classifier/detector, not a Planner or semantic authority source.

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
```

The semantic evaluator does not need to prove that:

```text
a formal Goal object exists
a formal next action exists
one exact Closure successor exists
Harness knows how to solve the current problem
```

## 9. Stop boundaries

At least the following stop conditions remain:

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

USER_INTERVENTION

SUBMISSION_UNCERTAIN
RUNTIME_UNSAFE

CONVERSATION_REBASE_REQUIRED

BUDGET_EXHAUSTED
```

Uncertainty may conservatively stop a Run. The design does not need to maximize continuation recall by guessing.

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
→ conversation rollover boundary
→ current ContinuationRun ends
→ Continuity workflow
→ C2
→ RESUME_ELIGIBLE
→ STOP
```

If the Human then chooses `Go ×N` on C2, that is a new bounded continuation authorization and a new Run.

At that point NOOS must still **not** ask for Goal again. The Continuity Checkpoint + BOOTSTRAP + Resume Verification path has already restored the current working position.

## 15. No cross-rollover budget carry

Old Run budget does not survive a binding change as actuation authority.

```text
old Run ENDED
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

Selecting `Go ×N` should start directly.

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
5. Accepted continuation #4 → `go` + lightweight mainline re-anchor.
6. Failed/uncertain send does not incorrectly advance the four-turn accepted-continuation counter.
7. Re-anchor does not introduce a new task or optional scope.
8. Assistant requests Human decision → Run stops before next `go`.
9. Assistant declares current mainline complete → Run stops.
10. CurrentConversationBinding changes → Run ends; no next `go`.
11. After successful rollover and `RESUME_ELIGIBLE`, a new Human `Go ×N` starts without another Goal prompt.

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

subject to the normal NOOS integration and independent review flow.
