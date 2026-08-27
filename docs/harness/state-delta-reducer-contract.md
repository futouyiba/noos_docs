# NOOS Harness｜State Delta + Reducer Contract v0

> **状态**：Design Baseline v0.1  
> **日期**：2026-08-27  
> **上位文档**：[Runtime Object Model & Authority Model v0](runtime-object-authority-model.md)  
> **目标**：定义 durable immutable Proposal 如何经过 durable Authorization，在一致事务边界内安全、可追溯、可并发检测且可幂等恢复地变成 Run State 变更。

---

## 0. 一句话说明

Harness 不能让 LLM 每隔几轮“重写一份最新 State”。

v0 的状态变化管线冻结为：

```text
Current State vN
      +
Durable Immutable Atomic Proposal
      ↓
Authority / Promotion Policy
      ↓
Durable AuthorizationResult
      ├─ requires_human → Durable PendingHumanGate
      ├─ denied
      └─ authorized
             ↓
       Authorized Delta
             ↓
           Reducer
             ↓
  crash-consistent local commit
       ├─ State vN+1
       ├─ ApplyResult
       └─ Transition / Audit Record
```

核心原则：

> **LLM proposes; Policy authorizes; Reducer applies; NOOS records.**

本文只解决“状态怎么安全变化”。它不决定下一步是否 Continue / Refresh / Rollover，也不负责 Provider dispatch / browser crash recovery。

但 **NOOS 自己的 State Apply 是否已经成功提交** 属于本文职责，而不是后续 Execution Journal 的 Provider-side recovery 职责。

---

# 1. 为什么不能让模型直接输出完整 State

Whole-state replacement 会带来静默遗忘、静默改义、权限越界和并发覆盖。因此 v0 使用 **operation-based Delta**。

---

# 2. Contract 中的四个主要对象

## 2.1 Proposal

Proposal 是 Worker / Reviewer / User / Source Observer 提出的**durable immutable 候选状态事务请求**。

```yaml
proposal:
  id: P-000123
  run_id: run-abc
  base_state_version: 17

  proposer:
    kind: agent
    ref: main-thread-worker

  operations: []
  operations_fingerprint: sha256:...
  created_at: 2026-08-27T10:00:00+08:00
```

Proposal 必须持久化，因为 AuthorizationResult / PendingHumanGate 都会引用它，Human approval 或 crash recovery 也需要重新读取原事务。

Proposal 创建后 immutable：如果 operations、payload、precondition 等会影响事务语义的内容改变，必须创建新的 Proposal ID / fingerprint。

> **Proposal 本身就是“请求”；Operation 描述期望发生的最终 State Transition。**

因此 v0 不使用 `propose_decision`、`propose_scope_change` 这类二次“提议操作”。

Agent 可以直接提交 `commit_decision`；这只表示请求，不表示 Agent 有权 commit。

## 2.2 AuthorizationResult

Policy 对**整个 Proposal 事务**给出 durable 结果：

```yaml
authorization_result:
  id: AZ-000087
  proposal_id: P-000123
  run_id: run-abc
  base_state_version: 17

  status: authorized | requires_human | denied
  authority_basis:
  gate_id:
  reason:
  created_at:
  supersedes_authorization_result_id:
```

## 2.3 Authorized Delta

只有 `AuthorizationResult.status = authorized` 时，Policy 才产生 Authorized Delta：

```yaml
authorized_delta:
  id: SD-000087
  proposal_id: P-000123
  authorization_result_id: AZ-000087
  run_id: run-abc
  base_state_version: 17
  proposal_operations_fingerprint: sha256:...
  operations: []
```

关键 invariant：

> **Authorized Delta 的 operation 有序列表及其有效 payload 必须与原 Proposal 完全一致，`proposal_operations_fingerprint` 必须匹配。Policy 只能 authorize / gate / deny，不能偷偷重写事务。**

如果 Human 或 Policy 想修改 transaction 内容，应生成新的 Proposal。

Authorized Delta 至少进入 transition/audit store。

## 2.4 ApplyResult

Reducer 处理 Authorized Delta 后返回并持久化：

```yaml
apply_result:
  id: AP-000087
  delta_id: SD-000087
  delta_fingerprint: sha256:...
  outcome: applied
  from_version: 17
  to_version: 18
  operation_results: []
  audit_record_id: AR-000087
```

`outcome`：

```text
applied
rejected_stale
rejected_precondition
rejected_invariant
no_op
```

ApplyResult 不创建 Human Gate。

对已经成功应用过的同一 `delta_id + fingerprint` 重试，Reducer/State Store **返回原 ApplyResult**，不重新执行事务，也不将其误判为 `rejected_stale`。

---

# 3. Proposal 是 v0 最小逻辑事务单元

一个逻辑变化：

```text
commit D-023
+
close Q-014 using D-023
+
move frontier to Q-015
```

形成一个 Proposal。

互不依赖的变化应拆成多个 Proposal。

v0 不引入 `atomic_group` / dependency graph / partial authorization。

> **Proposal atomicity → Authorization atomicity → Reducer atomicity。**

---

# 4. State 分区与写权限

```yaml
run_state:
  operational:
  committed:
  working:
  sources:
  meta:
```

## 4.1 Operational

包含 goal、deliverable、scope、phase、status。`scope` / `goal` 通常需要更高 authority。

## 4.2 Committed

```yaml
committed:
  constraints: []
  decisions: []
  rejected: []
```

这些是 Run 内正式治理状态。

## 4.3 Working

```yaml
working:
  hypotheses: []
  open_questions: []
  frontier:
  working_set:
  review_issues: []
```

Working State 可以给 agent 更大 delegated authority，但仍受 provenance / version / invariant 约束。

## 4.4 Sources

Run State 只保存 Hub Evidence / Source Store 中对象的引用：

```yaml
sources:
  source_ref_ids: []
  evidence_ref_ids: []
```

Reducer 不创建或修改 SourceRef/EvidenceRef 内容身份；只 attach 已存在的 reference IDs。

---

# 5. Operation Taxonomy v0

## 5.1 Working State

```text
add_hypothesis
update_hypothesis
retire_hypothesis
open_question
update_question
close_question
reopen_question
set_frontier
set_working_set
add_review_issue
resolve_review_issue
```

## 5.2 Committed State

```text
commit_constraint
supersede_constraint
commit_decision
supersede_decision
reopen_decision
commit_rejection
reopen_rejection
```

这些 operation 只描述 desired transition。

## 5.3 Operational

```text
set_phase
set_status
commit_scope_change
commit_goal_change
```

## 5.4 Source / Evidence attachment

```text
attach_source_ref
attach_evidence_ref
```

不提供：

```text
update_source_observation
record_source_supersession
```

新的 Source observation 及其 `supersedes_source_ref_id` 由 Hub Evidence / Source Store 创建时确定。

## 5.5 禁止通用绕过

```text
delete_anything
replace_state
set_arbitrary_path
raw_json_patch
```

---

# 6. Operation Envelope

```yaml
op_id: OP-001
op: close_question

target:
  id: Q-014

payload:
  resolution_ref: D-023

preconditions:
  - kind: target_status_is
    value: open

provenance:
  turn_refs: []
  evidence_ref_ids: []
  review_issue_refs: []

reason:
  summary: >
    D-023 已回答该问题。
```

Committed State 变更必须至少有一个可追溯依据。

---

# 7. Provenance Contract

## 7.1 Turn provenance

```yaml
turn_ref:
  provider_conversation_id:
  provider_message_ref:
  fingerprint:
  observed_at:
```

## 7.2 Source / Evidence provenance

```text
State Entity / Proposal provenance
      ↓
EvidenceRef
      ↓
Immutable SourceRef observation
```

EvidenceRef 不要求反向绑定一个已经存在的 State Claim，从而避免 pre-commit 循环依赖。

## 7.3 Review provenance

```yaml
review_issue_ref:
review_snapshot_id:
base_state_version:
```

Stale Review 由 Policy / Orchestrator 处理，不由 Reducer 猜测 rebase。

---

# 8. Source observation immutability

一旦 SourceRef 被引用，其内容身份不可原地覆盖。

```text
S-004 = Git commit A
Hub later observes commit B
→ create S-011
→ S-011.supersedes_source_ref_id = S-004
```

Run State 通过新的 Proposal attach S-011 / 对应 EvidenceRef；旧 S-004 保持不变。

---

# 9. Authority / Promotion Policy

Policy 可以内部逐 operation 检查，但 v0 正式输出是 **Proposal-level AuthorizationResult**。

## 9.1 事务级归并规则

```text
任一 denied
→ entire Proposal denied

else 任一 requires_human
→ entire Proposal requires_human

else
→ entire Proposal authorized
```

Operation-level evaluation 只作为 diagnostic，不能 partial execute。

## 9.2 Delegated authority

Agent 通常可自动获得 Working State 变更；goal/scope、supersede committed decision、reopen rejection、external irreversible write 等默认更严格。

## 9.3 Claim 类型影响 Authority

- preference / constraint / decision：用户或被授权 owner 通常高于 agent inference；
- fact：看 canonical/current EvidenceRef；
- inference：默认保持 hypothesis，除非 promotion。

---

# 10. Human Gate 的 durable ownership

Policy 持久化：

```yaml
authorization_result:
  id: AZ-000088
  proposal_id: P-000123
  run_id: run-abc
  base_state_version: 17
  status: requires_human
  gate_id: HG-004
  authority_basis: product_choice
  reason:
  created_at:
```

```yaml
pending_human_gate:
  id: HG-004
  run_id: run-abc
  proposal_id: P-000123
  authorization_result_id: AZ-000088
  gate_kind: product_choice
  prompt:
  status: pending | approved | rejected | cancelled
  created_at:
  resolved_at:
```

Human approval 后：

```text
AZ-000088 = requires_human
HG-004 = approved
        ↓
Policy re-evaluates immutable Proposal P-000123
        ↓
AZ-000091 = authorized
supersedes AZ-000088
        ↓
Authorized Delta with matching proposal_operations_fingerprint
```

若用户修改了提议内容，则必须创建新 Proposal，而不是沿用旧 fingerprint。

---

# 11. Versioning 与 Optimistic Concurrency

Proposal / Authorized Delta 都携带 `base_state_version`。

正常首次 apply 时：

```text
current_state.version != base_state_version
→ rejected_stale
```

但在判定 stale **之前**，State Store 必须先用 `delta_id` 查询是否已经存在该 Delta 的成功 Transition Record：

```text
same delta_id + same fingerprint + successful transition exists
→ return original ApplyResult
```

所以：

> **`rejected_stale` 只表示真正的 optimistic-concurrency 冲突，不表示“这笔 Delta 其实已成功，但上次调用方没拿到 receipt”。**

v0 不允许 Reducer 自动 merge 真正 stale 的 delta。

---

# 12. Preconditions

v0 支持少量确定性结构前提：

```text
target_exists
target_not_exists
target_status_is
field_equals
entity_version_is
committed_decision_active
question_open
rejected_item_active
```

语义性判断留给 Policy；Reducer 验证结构性前提。

---

# 13. Atomicity：三层一致

```text
Proposal atomic
Authorization atomic
Reducer apply atomic
```

一个 Proposal 只可能整个 denied、整个 requires_human、或整个 authorized 后 apply/no-op/reject。

---

# 14. Reducer 的 staged transaction semantics

Authorized Delta.operations 是**有序列表**。

例如：

```text
OP1 commit D-023
OP2 close Q-014 using D-023
OP3 move frontier to Q-015
```

Reducer 在 tentative/staged state 上按顺序执行：

```text
current_state v17
      ↓ clone / draft
tentative_state
      ↓ OP1
      ↓ OP2 sees D-023 from OP1
      ↓ OP3
      ↓ all valid?
      ├─ no → discard draft
      └─ yes → prepare durable transaction
```

v0 不支持引用尚未执行的未来 operation 结果；调用方必须按依赖顺序排列 operations。

---

# 15. Local Apply Crash Consistency 与 Delta Idempotency

这是 State Store 自己的 transaction contract，不属于 Provider Execution Journal。

## 15.1 Successful Apply 必须原子持久化三类结果

成功 apply 时，以下内容必须在**同一个 durable local transaction** 中提交：

```text
State mutation / new state version
+
ApplyResult
+
Transition / Audit Record keyed by delta_id
```

概念上：

```text
BEGIN LOCAL TRANSACTION

verify delta_id / fingerprint
verify current version = base_state_version
verify preconditions / invariants
apply staged mutation
write State v18
write ApplyResult AP-087
write Transition/Audit Record(delta_id = SD-087)

COMMIT
```

不能允许：

```text
State v18 durable
↓ crash
ApplyResult / Transition Record missing
```

否则重启后无法区分“同一 Delta 已经成功”与“另一笔事务推进了 State”。

## 15.2 `delta_id` 是 apply 幂等键

v0 冻结：

```text
same delta_id + same delta/proposal fingerprint
→ same logical apply
```

如果该 `delta_id` 已有成功 Transition Record：

```text
→ do not execute again
→ return the original persisted ApplyResult
```

从调用者角度，这是幂等重放，而不是新的 state transition。

## 15.3 ID reuse / tampering

```text
same delta_id + different fingerprint
→ rejected_invariant
```

`delta_id` 不允许被复用来承载另一笔内容不同的事务。

## 15.4 No-op 也要有 receipt

如果整笔 Authorized Delta 最终是 `no_op`，它虽然不 bump State version，但仍应 durable 记录 ApplyResult / Transition Record，保证同一 delta 重试仍返回同一个 `no_op` receipt，而不是重复进入业务判断。

## 15.5 与 Execution Journal 的边界

这里回答：

```text
NOOS local State Delta 到底 commit 了吗？
```

后续 Execution Journal 回答：

```text
prompt/browser/provider side effect 到底发生了吗？
```

两者都是幂等性问题，但 authority domain 不同，不能混为一个 journal。

---

# 16. Reducer Invariants v0

1. Committed Decision / Constraint / Rejection 不允许静默删除。
2. Rejected 必须通过显式 reopen path 才能复活。
3. Working ≠ Committed；Hypothesis promotion 必须显式请求并授权。
4. Run 内一等 State entity ID 唯一。
5. 引用完整性在 staged state 上验证；后置 operation 可引用前置 operation 新建 entity。
6. Scope / Goal 不可用通用字段写入绕过 Policy。
7. Reducer 只 attach SourceRef/EvidenceRef ID，不创建或修改 observation identity。
8. 至少一项实际 mutation 成功才 bump state version。
9. Authorized Delta 的 operations fingerprint 必须与原 immutable Proposal 匹配。
10. **State mutation、ApplyResult、Transition/Audit Record 必须 crash-consistent atomic commit。**
11. **`delta_id` 是 apply 幂等键；同 ID 同 fingerprint 重试返回原 ApplyResult。**
12. **同一 `delta_id` 若 fingerprint 不同，属于 invariant violation。**

---

# 17. Reducer 不应该做什么

Reducer 不做：LLM 语义判断、事实搜索、Authority/Human Gate、Evidence Store 写入、reviewer adjudication、stale auto-rebase、external side effect、next action。

Reducer 应尽量接近可单元测试的 pure-ish function；其 State Store adapter 负责 durable transaction / idempotent receipt。

---

# 18. Conflict / Stale / No-op

判定顺序必须避免把 successful replay 误报为 stale：

```text
1. delta_id 已成功应用？
   yes + fingerprint match → return original ApplyResult
   yes + fingerprint mismatch → rejected_invariant

2. current version != base_state_version
   → rejected_stale

3. precondition failure
   → rejected_precondition

4. invariant failure
   → rejected_invariant
```

No-op：

```text
所有 operation no-op
→ outcome = no_op
→ version unchanged
→ persist no_op ApplyResult / Transition Record
```

```text
至少一项实际 mutation
其余成功或 no-op
→ outcome = applied
→ version +1
```

---

# 19. AuthorizationResult 与 ApplyResult 边界

## Policy owns

```text
authorized
requires_human
denied
PendingHumanGate creation
```

## Reducer / State Store owns

```text
applied
no_op
rejected_stale
rejected_precondition
rejected_invariant
idempotent replay of prior ApplyResult
```

---

# 20. Review / stale Proposal

Reviewer Proposal 必须携带 `review_snapshot_id` / `base_state_version`。

Reducer/State Store 先检查同 `delta_id` 是否已经成功提交；若不是 replay 且 State 已推进，才返回 `rejected_stale`。后续是否重做 Review 由 Orchestrator / Policy 决定。

---

# 21. Compaction Worker 的输出

```text
Transcript Segment
      ↓
State Extraction
      ↓
Durable Immutable Atomic Proposal(s)
      ↓
Policy
      ↓
Reducer
```

互不依赖的变化拆成多个 Proposal；必须共同成立的 operations 放在同一 Proposal 并按依赖顺序排列。

---

# 22. Context Compiler 只消费 Applied State

Pending Proposal / PendingHumanGate 可以显式展示，但不能伪装成 Committed State。

---

# 23. State Delta v0 最小实现范围

```text
add_hypothesis
retire_hypothesis
open_question
close_question
set_frontier
commit_constraint
commit_decision
commit_rejection
supersede_decision
reopen_rejection
commit_scope_change
attach_source_ref
attach_evidence_ref
```

Source observation 创建/supersession 属于 Hub Evidence / Source Store。

---

# 24. 必须测试的场景

### A. Working mutation

`Proposal(add_hypothesis) → authorized → applied`

### B. Product Decision + crash-safe gate

```text
Proposal(commit_decision)
→ requires_human
→ durable Proposal + AuthorizationResult + PendingHumanGate
→ crash/restart
→ human approves
→ new AuthorizationResult authorized
→ apply / stale
```

### C. Atomic Proposal 一项需要 Human

整笔 Proposal requires_human，State 不 partial mutate。

### D. Same-transaction dependency

OP1 创建 D-023，OP2 在 staged state 中合法引用 D-023，最终一次 commit。

### E. Proposal tampering

```text
Authorization references P-123 fingerprint F1
Authorized Delta operations fingerprint F2
F1 != F2
→ rejected_invariant
```

### F. Stale reviewer

base v17 / current v20，且该 delta_id 未曾成功提交 → rejected_stale。

### G. Source re-observation

新 observation 创建新 SourceRef；Reducer 只 attach IDs。

### H. Mixed / all no-op

mixed mutation → applied + version+1；all no-op → no_op + version unchanged，并持久化 no-op receipt。

### I. Crash after local state commit boundary

在 durable transaction 的任意中间位置 crash：重启后只能看到“整笔未提交”或“State + ApplyResult + Transition Record 全部已提交”，不能看到半笔成功。

### J. Same Delta replay after lost caller receipt

```text
SD-087 已成功提交
调用方未收到 AP-087 / process crash
↓ retry SD-087
same delta_id + same fingerprint
→ return original AP-087
→ State version 不再次变化
```

### K. Delta ID collision / tampering

```text
SD-087 already recorded with F1
retry SD-087 with F2
→ rejected_invariant
```

---

# 25. 当前冻结结论

1. Proposal durable、immutable，本身就是 request。
2. Operation 是 desired final state transition；无 `propose_*` Reducer operation。
3. Proposal 是最小逻辑事务单元。
4. Proposal / Authorization / Reducer 使用同一事务边界。
5. AuthorizationResult durable；Human Gate 由 Policy durable 持有。
6. Authorized Delta 只在整笔 Proposal authorized 时存在，且必须匹配 Proposal operations fingerprint。
7. Reducer 使用 ordered staged state，再 atomic commit。
8. Reducer deterministic，不做 authority/LLM/source reasoning。
9. SourceRef observation immutable；Evidence Store owns object creation。
10. State/Proposal provenance 单向引用 EvidenceRef → SourceRef，避免 pre-commit 循环依赖。
11. Reducer 只 attach SourceRef/EvidenceRef IDs。
12. stale Proposal 不自动 merge。
13. 全部 no-op 不 bump version；mixed no-op + mutation 算 applied。
14. Context Compiler 只把 Applied/Committed State 当正式工作状态。
15. **Successful Apply 必须把 State mutation、ApplyResult、Transition/Audit Record 在同一 durable transaction 中提交。**
16. **`delta_id` 是 local apply 幂等键；同 ID 同 fingerprint 重试返回原 ApplyResult。**
17. **只有排除 successful replay 后，version mismatch 才能定义为 `rejected_stale`。**

---

# 26. 与下一层 Continuation 的接口

Continuation State Machine 只需要消费：

```text
Run State
AuthorizationResult
PendingHumanGate?
ApplyResult?
Runtime Signals
```

典型：

```text
AuthorizationResult = requires_human
→ PAUSED_HUMAN

AuthorizationResult = authorized
ApplyResult = applied / no_op / idempotent replay of prior success
→ DECIDE_NEXT_ACTION

ApplyResult = rejected_stale
→ 真正发生了并发/版本冲突
→ RECONCILE / REGENERATE
```

Continuation 不需要自行猜测“State vN+1 到底是不是这笔 Delta 造成的”；State Store 必须先通过 `delta_id` 幂等 receipt 把这个问题回答清楚。

---

## 当前状态

本文继续保持 **Design Baseline v0.1**。

本轮补齐 local State Apply crash consistency：State mutation、ApplyResult 与 Transition/Audit Record 同事务提交，`delta_id` 作为 apply 幂等键。至此 Reducer / State Store 对外提供的 `rejected_stale` 已经具有单义语义，可以安全交给 Continuation 消费。
