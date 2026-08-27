# NOOS Harness｜State Delta + Reducer Contract v0

> **状态**：Design Candidate v0  
> **日期**：2026-08-27  
> **上位文档**：[Runtime Object Model & Authority Model v0](runtime-object-authority-model.md)  
> **目标**：定义 durable Proposal 如何经过 durable Authorization，在一致事务边界内安全、可追溯、可并发检测地变成 Run State 变更。

---

## 0. 一句话说明

Harness 不能让 LLM 每隔几轮“重写一份最新 State”。

v0 的状态变化管线冻结为：

```text
Current State vN
      +
Durable Atomic Proposal
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

最简单的实现看似是：

```text
每 10 轮
→ 请模型输出最新 State YAML
→ 覆盖旧 State
```

但它会带来四类不可接受风险：

1. **静默遗忘**：旧 Decision / Constraint 被总结掉；
2. **静默改义**：旧结论被更流畅的新措辞悄悄改变；
3. **权限越界**：Hypothesis 被模型自己升级成 Committed Decision；
4. **并发覆盖**：Reviewer 基于 v17 的结果覆盖 Main 已推进到 v20 的状态。

因此 v0 使用 **operation-based Delta**，而不是 whole-state replacement。

---

# 2. Contract 中的四个主要对象

## 2.1 Proposal

Proposal 是 Worker / Reviewer / User / Source Observer 提出的**durable 候选状态事务请求**。

```yaml
proposal:
  id: P-000123
  run_id: run-abc
  base_state_version: 17

  proposer:
    kind: agent
    ref: main-thread-worker

  operations: []
  created_at: 2026-08-27T10:00:00+08:00
```

Proposal 必须持久化，因为 AuthorizationResult / PendingHumanGate 都会引用它，Human approval 或 crash recovery 也需要重新读取原事务。

Proposal 不代表 State 已改变，也不代表 proposer 拥有执行这些 operation 的 authority。

最重要的语义：

> **Proposal 本身就是“请求”；Operation 描述期望发生的最终 State Transition。**

因此 v0 不使用 `propose_decision`、`propose_scope_change` 这类二次“提议操作”。

Agent 可以直接提交：

```yaml
operations:
  - op: commit_decision
```

这只表示“请求 committed state 出现这个 decision”，并不表示 Agent 有权 commit。

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

AuthorizationResult 是 Policy 的正式持久化输出。

## 2.3 Authorized Delta

只有 `AuthorizationResult.status = authorized` 时，Policy 才产生 Authorized Delta：

```yaml
authorized_delta:
  id: SD-000087
  proposal_id: P-000123
  authorization_result_id: AZ-000087
  run_id: run-abc
  base_state_version: 17
  operations: []
```

Authorized Delta 与 Proposal 具有相同事务边界，不允许 Policy 从一个 Proposal 中只摘取部分 operation 继续执行。

Authorized Delta 至少应进入 transition/audit store，以便恢复和解释 State transition。

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

`outcome` 至少允许：

```text
applied
rejected_stale
rejected_precondition
rejected_invariant
no_op
```

ApplyResult 不创建 Human Gate。Human Gate 属于 Policy / Authorization 边界。

---

# 3. Proposal 是 v0 最小逻辑事务单元

例如一个逻辑变化：

```text
commit D-023
+
close Q-014 using D-023
+
move frontier to Q-015
```

应该形成**一个 Proposal**。

而两个互不依赖的变化：

```text
add hypothesis H-9
```

与：

```text
request a product decision requiring human authority
```

应该形成**两个 Proposal**。

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

例如 goal、deliverable、scope、phase、status。

`scope` / `goal` 通常需要更高 authority；`phase` 可以由 delegated policy 自动推进。

## 4.2 Committed

```yaml
committed:
  constraints: []
  decisions: []
  rejected: []
```

这些是 Run 内正式提交的治理状态。任何新增、supersede、reopen 都必须经过 Authority / Promotion Policy。

## 4.3 Working

```yaml
working:
  hypotheses: []
  open_questions: []
  frontier:
  working_set:
  review_issues: []
```

Working State 可以给 agent 更大的 delegated authority，但仍必须满足 provenance / version / invariant。

## 4.4 Sources

Run State 只保存 Hub Evidence / Source Store 中对象的引用：

```yaml
sources:
  source_ref_ids: []
  evidence_ref_ids: []
```

`SourceRef` 表示“观察了什么”；`EvidenceRef` 表示“这个 observation 对哪个 Claim 起什么证据作用”。

Reducer 不创建或修改 SourceRef/EvidenceRef 的内容身份；它只 attach 已存在的 immutable reference IDs。

---

# 5. Operation Taxonomy v0

v0 不追求通用 JSON Patch。Operation 必须带领域语义。

## 5.1 Working State operations

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

## 5.2 Committed State operations

```text
commit_constraint
supersede_constraint

commit_decision
supersede_decision
reopen_decision

commit_rejection
reopen_rejection
```

这些 operation 在 Proposal 中出现，只表示“请求该最终 transition”。

## 5.3 Operational operations

```text
set_phase
set_status
commit_scope_change
commit_goal_change
```

同样，`commit_scope_change` 是 desired transition，不代表 proposer 拥有 scope authority。

## 5.4 Source / Evidence attachment operations

v0 Reducer 只负责把已存在的证据对象挂到 Run State：

```text
attach_source_ref
attach_evidence_ref
```

不提供：

```text
update_source_observation
record_source_supersession
```

新的 Source observation 及其 `supersedes_source_ref_id` 由 Hub Evidence / Source Store 在创建新 SourceRef 时确定；Reducer 不改写历史 observation。

## 5.5 明确禁止的通用操作

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

Provenance 需要回答：

- 这个 Decision 从哪里来？
- 为什么 Q-014 被关闭？
- Reviewer 基于哪个 snapshot 提出 blocker？
- factual claim 依据哪个 source observation？

## 7.1 Turn provenance

```yaml
turn_ref:
  provider_conversation_id:
  provider_message_ref:
  fingerprint:
  observed_at:
```

## 7.2 Source / Evidence provenance

事实性 Claim 优先引用 EvidenceRef：

```text
Claim / State Entity
      ↓
EvidenceRef
      ↓
Immutable SourceRef observation
```

## 7.3 Review provenance

```yaml
review_issue_ref:
review_snapshot_id:
base_state_version:
```

若 Review Snapshot 已 stale，Policy 可拒绝或要求重新生成 Proposal；Reducer 不猜测 rebase。

---

# 8. Source observation immutability

一旦 SourceRef 被 EvidenceRef / Decision provenance 引用，它的内容身份字段不可原地覆盖。

例如：

```text
S-004 = Git commit A, observed_at T1, fingerprint F1
```

后来 Git 已到 commit B：

```text
Hub Evidence Store creates S-011
S-011 = commit B, observed_at T2, fingerprint F2
S-011.supersedes_source_ref_id = S-004
```

而不是修改 S-004。

Run State 如需使用新证据，只通过新的 Proposal `attach_source_ref / attach_evidence_ref` 挂入新 ID。

---

# 9. Authority / Promotion Policy

Reducer 不负责判断“谁说了算”。

Policy 的输入至少包括：

```yaml
proposal:
current_state:
authority_policy:
proposer_identity:
evidence_metadata:
```

Policy 可以在内部逐 operation 检查权限与条件，但 **v0 的正式输出是 Proposal-level AuthorizationResult**。

## 9.1 事务级 Authorization 归并规则

若内部检查存在任一：

```text
denied
```

→ 整个 Proposal `denied`。

否则若存在任一：

```text
requires_human
```

→ 整个 Proposal `requires_human`。

只有全部 operation 均可授权：

```text
authorized
```

→ 才生成 Authorized Delta。

Operation-level evaluation 只作为 diagnostic，不能把其余 operation 偷偷拆出去执行。

## 9.2 v0 推荐 delegated authority

Agent 默认可请求并通常自动获得：

```text
add/update hypothesis
open exploratory question
set frontier
attach source/evidence ref
update working set
```

Agent 默认不能自动获得：

```text
change goal/scope
supersede committed decision
revive rejected approach without reopen path
external irreversible write
```

`close_question` 是否 delegated 取决于 question 类型。

## 9.3 Claim 类型影响 Authority

- `preference / constraint / decision`：用户或被授权 owner 的明确表达通常高于 agent inference；
- `fact`：依据对应领域 canonical/current EvidenceRef，而不是固定 `user > document`；
- `inference`：默认保持 working hypothesis，除非走 promotion。

---

# 10. Human Gate 的 durable ownership

`requires_human` 发生在 Policy 阶段，不属于 Reducer ApplyResult。

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

以及：

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

这样即使 Browser / Hub 在用户回答前崩溃，Run 仍知道有一个未解决的 authority boundary，并且可以重新读取原 durable Proposal。

## 10.1 Human approval 后如何继续

```text
AZ-000088 = requires_human
HG-004 = approved
        ↓
Policy re-evaluates durable Proposal P-000123
        ↓
AZ-000091 = authorized
supersedes AZ-000088
        ↓
Authorized Delta
```

不要把旧 AuthorizationResult 原地改成 authorized。

如果等待期间 `base_state_version` 已过期，最终 Reducer 可以返回 `rejected_stale`；是否重新生成 Proposal 留给上层。

---

# 11. Versioning 与 Optimistic Concurrency

Proposal / Authorized Delta 都携带：

```yaml
base_state_version: N
```

Reducer 只在：

```text
current_state.version == base_state_version
```

时直接 apply，否则 `rejected_stale`。

v0 **不允许 Reducer 自动 merge stale delta**。

---

# 12. Preconditions

`base_state_version` 防整体 stale；operation precondition 表达局部结构前提。

v0 支持少量确定性类型：

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

如果“reopen condition 是否真的满足”需要语义判断，应在 Policy 阶段完成；Reducer 只验证结构性前提和已授权 transaction。

---

# 13. Atomicity：三层一致

v0 冻结：

```text
Proposal atomic
Authorization atomic
Reducer apply atomic
```

一个 Proposal 只可能：

```text
整个 denied
整个 requires_human
整个 authorized → 整个 apply/no-op/reject
```

若调用方有独立 mutation，调用方负责拆成多个 Proposal。

---

# 14. Reducer 的 staged transaction semantics

All-or-Nothing 还不足以解释同一事务内的内部引用。

例如：

```text
OP1 commit D-023
OP2 close Q-014 using D-023
OP3 move frontier to Q-015
```

`D-023` 在事务开始前不存在，但 OP2 合法依赖 OP1。

因此 v0 冻结：

> **Authorized Delta.operations 是有序列表；Reducer 在 tentative/staged state 上按顺序执行和校验，只有所有 operation 成功后才一次性 commit staged state。**

伪流程：

```text
current_state v17
      ↓ clone / draft
tentative_state
      ↓ OP1
      ↓ OP2 sees result of OP1
      ↓ OP3 sees prior staged mutations
      ↓ all valid?
      ├─ no → discard tentative state
      └─ yes → commit as v18
```

v0 不支持引用“尚未执行的未来 operation”结果。调用方必须按依赖顺序排列 operations。

这样既允许事务内引用，又保持最终原子性。

---

# 15. Reducer Invariants v0

Reducer 必须是确定性、无模型推理的状态转换器。

## 15.1 不允许静默删除 Committed State

Committed Decision / Constraint / Rejection 不能直接消失，只能显式 supersede/reopen，并保留历史与 provenance。

## 15.2 Rejected 不允许直接复活

必须通过 `reopen_rejection`，并引用 reopen reason/evidence。

## 15.3 Working ≠ Committed

`add_hypothesis` 不能产生 committed decision。提升时 Proposal 使用 `commit_decision`，Policy 决定是否授权。

## 15.4 ID 唯一

所有一等 State entity ID 在 Run 内必须唯一。

## 15.5 引用完整

引用完整性在 staged state 上验证；后置 operation 可以引用前置 operation 刚创建的 entity。

## 15.6 Scope / Goal 不能被通用字段写入绕过

不存在 `set_field` 路径绕过 `commit_scope_change` / `commit_goal_change` 与 Policy。

## 15.7 Evidence provenance 不可被 observation overwrite

Reducer 只 attach SourceRef/EvidenceRef ID，不创建或修改 observation identity。

## 15.8 Version 单调递增

至少一项实际 mutation 成功时 `vN → vN+1`；失败或全部 no-op 不产生新版本。

---

# 16. Reducer 不应该做什么

Reducer **不做**：

- LLM 语义判断；
- 事实搜索；
- Authority / Human Gate 判断；
- SourceRef/EvidenceRef 内容创建或改写；
- 自动裁决 reviewer 冲突；
- 自动 rebase stale Proposal；
- external side effect；
- next action；
- PendingHumanGate creation。

Reducer 应尽量接近一个可单元测试的 pure-ish function。

---

# 17. Conflict / Stale / No-op 语义

## 17.1 Stale

`base_state_version != current_state_version` → `rejected_stale`。

## 17.2 Precondition failure

局部结构前提不成立 → `rejected_precondition`。

## 17.3 Invariant failure

引用非法、违反 committed-state invariant 等 → `rejected_invariant`。

## 17.4 No-op

```text
所有 operation 都是 no-op
→ outcome = no_op
→ state_version 不变
```

```text
至少一项实际 mutation
且其余 operation 均成功或 no-op
→ outcome = applied
→ state_version + 1
```

`no-op` 不视为事务失败。

---

# 18. AuthorizationResult 与 ApplyResult 的接口边界

## AuthorizationResult — Policy owns

```text
authorized
requires_human
denied
PendingHumanGate creation
```

## ApplyResult — Reducer owns

```text
applied
no_op
rejected_stale
rejected_precondition
rejected_invariant
```

Continuation Runtime 以后消费：

```text
Run State
+ latest AuthorizationResult / PendingHumanGate
+ ApplyResult（若 Reducer执行过）
+ Runtime Signals
```

---

# 19. Review / stale Proposal

Reviewer Proposal 必须携带：

```yaml
review_snapshot_id:
base_state_version:
```

若主线程已推进，Reducer `rejected_stale`；是否重新生成 Proposal 由 Review Orchestrator / Policy 决定。

---

# 20. Compaction Worker 的输出

Compaction Worker 不拥有 State。

```text
Transcript Segment
      ↓
State Extraction
      ↓
Durable Atomic Proposal(s)
      ↓
Policy
      ↓
Reducer
```

互不依赖的变化拆成多个 Proposal；必须共同成立的 operations 放在同一 Proposal，并按依赖顺序排列。

禁止 whole-state overwrite。

---

# 21. Context Compiler 只消费 Applied State

Pending Proposal / PendingHumanGate 可以显式展示，但不能伪装成 Committed State。

---

# 22. State Delta v0 的最小实现范围

M0/M1 可先实现：

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

Source observation 的创建/supersession 由 Hub Evidence / Source Store 负责，不属于 Reducer operation。

---

# 23. 必须测试的场景

### A. 普通 Working mutation

```text
Proposal(add_hypothesis)
→ authorized
→ applied
```

### B. Product Decision + crash-safe gate

```text
Proposal(commit_decision)
→ requires_human
→ durable Proposal + AuthorizationResult + PendingHumanGate
→ crash/restart
→ gate 和原 Proposal 仍存在
→ human approves
→ new AuthorizationResult authorized
→ applied / stale
```

### C. Atomic Proposal 中一项需要 Human

```text
commit D-023
close Q-014
set frontier
```

若 `commit D-023` requires_human：

```text
整个 Proposal requires_human
State 不发生部分 mutation
```

### D. Same-transaction forward dependency

```text
OP1 commit D-023
OP2 close Q-014 using D-023
```

Reducer 在 staged state 中执行 OP1 后，OP2 可以合法看到 D-023；最终一次性 commit。

### E. Stale reviewer

```text
Proposal base v17
Current v20
→ rejected_stale
```

### F. Source re-observation

```text
S-004 = commit A
Hub observes commit B
→ Hub creates S-011 superseding S-004
→ Reducer only attaches S-011 / new EvidenceRef IDs
→ S-004 remains immutable
```

### G. Mixed no-op

```text
OP1 no-op
OP2 actual mutation
→ outcome applied
→ version +1
```

### H. All no-op

```text
all operations no-op
→ outcome no_op
→ version unchanged
```

---

# 24. 当前冻结结论

1. **Proposal durable，且本身就是 request。**
2. **Operation 是 desired final state transition；v0 不存在 `propose_*` Reducer operation。**
3. **Proposal 是最小逻辑事务单元。**
4. **Proposal / Authorization / Reducer 使用同一事务边界。**
5. **AuthorizationResult durable。**
6. **Human Gate 由 Policy 创建并持久化，不由 Reducer产生。**
7. **Authorized Delta 只在整笔 Proposal authorized 时存在。**
8. **Reducer 使用 ordered staged state，再 atomic commit。**
9. **Reducer deterministic，不做 authority/LLM/source reasoning。**
10. **SourceRef observation immutable；Source/Evidence object creation 属于 Hub Evidence Store。**
11. **Reducer 只 attach SourceRef/EvidenceRef IDs。**
12. **stale Proposal 不自动 merge。**
13. **全部 no-op 不 bump version；mixed no-op + mutation 算 applied。**
14. **Context Compiler 只把 Applied/Committed State 当正式工作状态。**

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
```

```text
AuthorizationResult = authorized
ApplyResult = applied
→ DECIDE_NEXT_ACTION
```

```text
ApplyResult = rejected_stale
→ RECONCILE / REGENERATE
```

---

## 当前状态

本文继续保持 **Design Candidate v0**，等待与 Runtime Object & Authority Model 的接口对审结论。

本轮已关闭：

- Proposal vs `propose_*` 双重建模；
- per-operation authorization 与 all-or-nothing atomicity 冲突；
- `requires_human` 缺少 durable owner；
- Proposal 在 Human Gate / crash 中不可恢复；
- Source observation 可被原地覆盖；
- Reducer 与 Hub Evidence Store ownership 混淆；
- 同事务内前置创建/后置引用的 staged-state 语义；
- mixed no-op 语义空洞。
