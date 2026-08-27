# NOOS Harness｜State Delta + Reducer Contract v0

> **状态**：Design Baseline v0.1  
> **日期**：2026-08-27  
> **上位文档**：[Runtime Object Model & Authority Model v0](runtime-object-authority-model.md)  
> **目标**：定义 durable immutable Proposal 如何经过 durable Authorization，在一致事务边界内安全、可追溯、可并发检测地变成 Run State 变更。

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
        State vN+1
             +
         ApplyResult
             +
         Audit Record
```

核心原则：

> **LLM proposes; Policy authorizes; Reducer applies; NOOS records.**

本文只解决“状态怎么安全变化”。它不决定下一步是否 Continue / Refresh / Rollover，也不负责 Provider dispatch / crash recovery。

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

Stale Review 由 Policy / Orchestrator 处理，不由 Reducer猜测 rebase。

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

```text
current_state.version != base_state_version
→ rejected_stale
```

v0 不允许 Reducer 自动 merge stale delta。

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
      └─ yes → atomic commit as v18
```

v0 不支持引用尚未执行的未来 operation 结果；调用方必须按依赖顺序排列 operations。

---

# 15. Reducer Invariants v0

1. Committed Decision / Constraint / Rejection 不允许静默删除。
2. Rejected 必须通过显式 reopen path 才能复活。
3. Working ≠ Committed；Hypothesis promotion 必须显式请求并授权。
4. Run 内一等 State entity ID 唯一。
5. 引用完整性在 staged state 上验证；后置 operation 可引用前置 operation 新建 entity。
6. Scope / Goal 不可用通用字段写入绕过 Policy。
7. Reducer 只 attach SourceRef/EvidenceRef ID，不创建或修改 observation identity。
8. 至少一项实际 mutation 成功才 bump state version。
9. Authorized Delta 的 operations fingerprint 必须与原 immutable Proposal 匹配。

---

# 16. Reducer 不应该做什么

Reducer 不做：LLM 语义判断、事实搜索、Authority/Human Gate、Evidence Store 写入、reviewer adjudication、stale auto-rebase、external side effect、next action。

Reducer 应尽量接近可单元测试的 pure-ish function。

---

# 17. Conflict / Stale / No-op

```text
stale → rejected_stale
precondition failure → rejected_precondition
invariant failure → rejected_invariant
```

```text
所有 operation no-op
→ outcome = no_op
→ version unchanged
```

```text
至少一项实际 mutation
其余成功或 no-op
→ outcome = applied
→ version +1
```

---

# 18. AuthorizationResult 与 ApplyResult 边界

## Policy owns

```text
authorized
requires_human
denied
PendingHumanGate creation
```

## Reducer owns

```text
applied
no_op
rejected_stale
rejected_precondition
rejected_invariant
```

---

# 19. Review / stale Proposal

Reviewer Proposal 必须携带 `review_snapshot_id` / `base_state_version`。State 已推进时，Reducer `rejected_stale`；后续是否重做 Review 由 Orchestrator / Policy 决定。

---

# 20. Compaction Worker 的输出

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

# 21. Context Compiler 只消费 Applied State

Pending Proposal / PendingHumanGate 可以显式展示，但不能伪装成 Committed State。

---

# 22. State Delta v0 最小实现范围

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

# 23. 必须测试的场景

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

base v17 / current v20 → rejected_stale。

### G. Source re-observation

新 observation 创建新 SourceRef；Reducer 只 attach IDs。

### H. Mixed / all no-op

mixed mutation → applied + version+1；all no-op → no_op + version unchanged。

---

# 24. 当前冻结结论

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

---

# 25. 与下一层 Continuation 的接口

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
ApplyResult = applied
→ DECIDE_NEXT_ACTION

ApplyResult = rejected_stale
→ RECONCILE / REGENERATE
```

---

## 当前状态

本文升为 **Design Baseline v0.1**。

经过与 Runtime Object & Authority Model 的接口对审，Proposal durability/immutability、Authorization ownership、事务原子性、staged Reducer、Source/Evidence ownership、provenance 与 Human Gate 边界已经互相闭合。后续若实现/Eval 暴露问题，通过显式版本演进调整，而不是继续在进入 Continuation 前无限打磨。
