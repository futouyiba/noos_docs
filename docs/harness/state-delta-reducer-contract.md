# NOOS Harness｜State Delta + Reducer Contract v0

> **状态**：Design Candidate v0  
> **日期**：2026-08-27  
> **上位文档**：[Runtime Object Model & Authority Model v0](runtime-object-authority-model.md)  
> **目标**：定义 Proposal 如何在 Authority Policy 控制下，安全、可追溯、可并发检测地变成 Run State 变更。

---

## 0. 一句话说明

Harness 不能让 LLM 每隔几轮“重写一份最新 State”。

真正需要的是：

```text
Current State vN
      +
Proposal Delta
      ↓
Authority / Promotion Policy
      ↓
Authorized Delta
      ↓
Reducer
      ↓
State vN+1
      +
Audit Record
```

核心原则：

> **LLM proposes; Policy authorizes; Reducer applies; NOOS records.**

这篇 Contract 只解决“状态怎么安全变化”。它不负责决定下一步是否 Continue / Refresh / Rollover，也不负责 Provider dispatch/recovery。

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
3. **权限越界**：Hypothesis 被模型自己升级成 Confirmed Decision；
4. **并发覆盖**：Reviewer 基于 v17 的结果覆盖 Main 已推进到 v20 的状态。

因此 v0 采用 operation-based Delta，而不是 whole-state replacement。

---

# 2. Contract 中的三个不同对象

## 2.1 Proposal

Proposal 是 Worker / Reviewer / User / Source Observer 提出的**候选变更请求**。

它可以包含调用方没有 authority 执行的操作。

```yaml
proposal:
  proposal_id: P-000123
  run_id: run-abc
  base_state_version: 17
  proposer:
    kind: agent
    ref: main-thread-worker
  operations: []
  created_at: 2026-08-27T10:00:00+08:00
```

Proposal 不代表 State 已改变。

## 2.2 Authorized Delta

Authority / Promotion Policy 对 Proposal 逐项判断后，产生 Authorized Delta。

```yaml
authorized_delta:
  delta_id: SD-000087
  proposal_id: P-000123
  run_id: run-abc
  base_state_version: 17
  operations: []
  authorization_record: []
```

只有 Authorized Delta 才能送入 Reducer。

## 2.3 Apply Result

Reducer 处理后返回：

```yaml
apply_result:
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

v0 不做“部分成功”。见第 10 节原子性。

---

# 3. State 分区与写权限

基于 Runtime Object Model：

```yaml
run_state:
  operational:
  committed:
  working:
  sources:
  meta:
```

不同分区不是同等权限。

## 3.1 Operational

例如：

- goal；
- deliverable；
- scope；
- phase；
- status。

其中 `scope` / `goal` 通常需要更高 authority；`phase` 可被 delegated policy 自动推进。

## 3.2 Committed

```yaml
committed:
  constraints: []
  decisions: []
  rejected: []
```

这些是 Run 内正式提交的治理状态。

任何新增、supersede、reopen 都必须经过 Promotion / Authority Policy。

## 3.3 Working

```yaml
working:
  hypotheses: []
  open_questions: []
  frontier:
  working_set:
  review_issues: []
```

Working State 可以给 agent 更大 delegated authority，但仍必须满足 provenance / version / invariant。

## 3.4 Sources

Source Ref 是引用对象，不是把外部事实永久复制为 Truth。

新增 source 时应使用上位文档的正交维度：

```yaml
origin_kind:
authority_role:
temporal_status:
claim_kind:
```

---

# 4. Operation Taxonomy v0

v0 不追求通用 JSON Patch。我们需要的是带领域语义的 operation。

## 4.1 Working State operations

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

## 4.2 Committed State operations

```text
propose_constraint
commit_constraint
supersede_constraint

propose_decision
commit_decision
supersede_decision
reopen_decision

commit_rejection
reopen_rejection
```

注意：Worker 通常只能产生 `propose_*`，能否转成 `commit_*` 由 Policy 决定。

## 4.3 Operational operations

```text
set_phase
set_status
propose_scope_change
commit_scope_change
propose_goal_change
commit_goal_change
```

## 4.4 Source operations

```text
attach_source_ref
update_source_observation
mark_source_stale
```

## 4.5 不允许的通用操作

v0 明确不提供：

```text
delete_anything
replace_state
set_arbitrary_path
raw_json_patch
```

因为这些操作绕过语义 invariant，容易重新制造“模型重写整个状态”的风险。

---

# 5. Operation Envelope

每个 operation 都应有统一 envelope：

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
  source_refs: []
  review_issue_refs: []

reason:
  summary: >
    D-023 已回答该问题。
```

并非所有 operation 都要求所有 provenance 类型，但 Committed State 变更必须至少有一个可追溯依据。

---

# 6. Provenance Contract

## 6.1 Provenance 的目的

不是为了“把所有聊天都复制进 State”，而是能回答：

- 这个 Decision 从哪里来的？
- 为什么 Q-014 被关闭？
- Reviewer 基于哪个 snapshot 提出 blocker？
- 这个事实 claim 依据哪个 source/version？

## 6.2 Turn provenance

```yaml
turn_ref:
  provider_conversation_id:
  provider_message_ref:
  fingerprint:
  observed_at:
```

## 6.3 Source provenance

```yaml
source_ref_id:
claim_excerpt_fingerprint:
observed_at:
```

Source Ref 自身包含：

```yaml
origin_kind:
authority_role:
temporal_status:
claim_kind:
version:
freshness:
```

## 6.4 Review provenance

```yaml
review_issue_ref:
review_snapshot_id:
base_state_version:
```

如果 Review Snapshot 已 stale，Policy 可拒绝或要求 rebase，不应由 Reducer猜测。

---

# 7. Authority / Promotion Policy 如何作用到 Delta

Reducer 不负责判断“谁说了算”。

Policy 的输入至少包括：

```yaml
proposal:
current_state:
authority_policy:
proposer_identity:
source_metadata:
```

输出是每个 operation 的 authorization decision：

```yaml
authorization_record:
  - op_id: OP-001
    decision: authorized
    authority_basis: delegated_agent_working_state

  - op_id: OP-002
    decision: requires_human
    authority_basis: product_choice

  - op_id: OP-003
    decision: denied
    authority_basis: stale_review_snapshot
```

## 7.1 v0 的推荐 delegated authority

Agent 默认可：

```text
add/update hypothesis
open exploratory question
set frontier
attach evidence/source ref
update working set
```

Agent 默认不可自行：

```text
change goal/scope
supersede committed decision
revive rejected approach without reopen path
perform external irreversible write
```

`close_question` 是否可 delegated 取决于 question 类型：纯 exploratory question 可允许；涉及 product choice / scope / committed decision 的问题应 Human Gate。

## 7.2 Claim kind 影响 Source Authority

Policy 处理事实 claim 时必须先判断 claim kind：

- `preference / constraint / decision`：用户明确表达通常高于 agent inference；
- `fact`：应看领域 canonical/current source，而不是固定 `user > document`；
- `inference`：必须保持为 inference/hypothesis，除非走 promotion。

---

# 8. Versioning 与 Optimistic Concurrency

每个 Proposal / Delta 都必须携带：

```yaml
base_state_version: N
```

Reducer 应只在：

```text
current_state.version == base_state_version
```

时直接 apply。

否则返回：

```text
rejected_stale
```

或者交给上层做显式 rebase。

v0 **不允许 Reducer 自动 merge stale delta**。

原因是：

- 同时修改 frontier 可能冲突；
- question 状态可能已经变化；
- reviewer issue 可能已被主线程解决；
- committed decision 可能已被 supersede。

自动 merge 应作为后续独立能力，不进入 v0。

---

# 9. Preconditions

`base_state_version` 只能防整体 stale，不能表达 operation 的局部语义前提。

所以 operation 可以声明 preconditions。

v0 建议支持少量确定性类型：

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

例如：

```yaml
op: reopen_decision
preconditions:
  - kind: committed_decision_active
    target_id: D-017
  - kind: reopen_condition_satisfied
    evidence_ref: E-009
```

其中 `reopen_condition_satisfied` 若需要语义判断，应在 Policy 阶段得到授权结论；Reducer 只验证授权结果与结构前提。

---

# 10. 原子性：v0 采用 All-or-Nothing

一个 Authorized Delta 中如果有多个 operation：

```text
OP1
OP2
OP3
```

v0 默认事务语义：

> **任何一个 operation 失败，整个 Delta 不修改 State。**

原因：

一个逻辑变更经常跨多个对象，例如：

```text
commit D-023
+
close Q-014 using D-023
+
move frontier to Q-015
```

若只成功一半，State 会进入难以理解的中间态。

未来若性能或大批量 source ingest 需要 partial apply，可单独定义 batch contract；不要污染核心 State Delta。

---

# 11. Reducer Invariants v0

Reducer 必须是确定性、无模型推理的状态转换器。

至少维护以下 invariant。

## 11.1 不允许静默删除 Committed State

Committed Decision / Constraint / Rejection 不能直接消失。

只能显式：

```text
active → superseded
active → reopened
```

并保留历史记录与 provenance。

## 11.2 Rejected 不允许直接复活

必须通过 `reopen_rejection`，并引用 reopen reason/evidence。

## 11.3 Working ≠ Committed

`add_hypothesis` 不能产生 committed decision。

必须存在显式 promotion operation。

## 11.4 ID 唯一

所有一等 State entity ID 在 Run 内必须唯一。

## 11.5 引用完整

`close_question.resolution_ref` 必须指向存在且合法的 resolution entity。

## 11.6 Scope/Goal 不能被低权限 operation 绕过

不存在通用 `set_field` 路径可以修改 scope/goal。

## 11.7 Source claim 不能伪装成永真事实

事实性 committed claim 若依赖可变 source，应保留 Source Ref / observation provenance；Source stale 不一定自动撤销 Decision，但必须允许上层识别事实依据已过期。

## 11.8 Version 单调递增

成功 apply：

```text
vN → vN+1
```

失败/no-op 不产生虚假的 State version。

---

# 12. Reducer 不应该做什么

Reducer **不做**：

- LLM 语义判断；
- 事实搜索；
- 判断用户“真正想要什么”；
- 自动裁决 reviewer 冲突；
- 自动 rebase stale proposal；
- 执行外部 side effect；
- 自动决定 next action。

这些分别属于 Policy、Source Resolver、Review Orchestrator、Continuation Runtime、Execution subsystem。

这条边界很重要：Reducer 应尽量接近一个可单元测试的纯函数。

---

# 13. Conflict / Stale / No-op 语义

## 13.1 Stale

```text
base_state_version != current_state_version
```

→ `rejected_stale`

上层可以：

- 重新生成 Proposal；
- rebase reviewer issue；
- 请求人处理。

## 13.2 Precondition failure

State version 没变，但目标语义不再满足。

→ `rejected_precondition`

## 13.3 Invariant failure

例如删除 committed decision、引用不存在对象。

→ `rejected_invariant`

## 13.4 No-op

例如重复 attach 同一个 source ref，且 fingerprint/version 未变化。

→ `no_op`

No-op 不应该生成新 state version。

---

# 14. 三个典型例子

## 14.1 Agent 添加 Hypothesis

Proposal：

```yaml
base_state_version: 17
operations:
  - op_id: OP-101
    op: add_hypothesis
    payload:
      id: H-021
      claim: Feeding Motivation 可能更适合作为 derived working state。
    provenance:
      turn_refs: [T-188]
```

Policy：agent 对 Working State 有 delegated authority。

Reducer：验证 ID / version / provenance，apply。

结果：

```text
State v18
H-021 exists in working.hypotheses
```

## 14.2 Agent 试图直接提交产品 Decision

Proposal：

```yaml
op: commit_decision
payload:
  id: D-031
  claim: Secondary 必须固定为 Primary 的 0.7。
```

Policy 检测：这是产品选择，当前 policy `human_required`。

结果：

```text
requires_human
```

Reducer 根本不会看到这条未经授权的 operation。

## 14.3 Reviewer 基于旧 Snapshot 返回 Blocker

Reviewer Proposal：

```yaml
base_state_version: 37
review_snapshot_id: RS-008
op: add_review_issue
```

当前 State 已是 v44。

结果：

```text
rejected_stale
```

上层 Review Orchestrator 决定是否：

```text
rebase issue against v44
rerun reviewer
ignore as resolved
```

Reducer 不猜。

---

# 15. Compaction 与 State Delta 的关系

Stateful Compaction 不应该输出一份完整“最新 State”。

它应该产生：

```text
1. Proposal Delta
2. Carry Context / Working Set update
3. 可选 Checkpoint recommendation
```

例如：

```text
Recent Transcript Segment
        ↓
Compaction Worker
        ↓
Proposal:
- add hypothesis
- close question
- propose decision
- set frontier
        ↓
Policy
        ↓
Authorized Delta
        ↓
Reducer
```

因此 Compaction Worker 永远不是 State owner。

---

# 16. Context Compiler 只读取已应用 State

Context Compiler 不应该直接消费“尚未授权的 Proposal”作为 authoritative state。

默认顺序：

```text
Proposal
→ Policy
→ Reducer
→ State vN+1
→ Context Compiler
→ Context Projection
```

如果某个 pending Human Gate 需要展示 candidate decision，可以显式放入 `pending proposals` 区，而不能伪装成 committed state。

---

# 17. 与 Continuation State Machine 的接口

State Delta Contract 不决定下一步 Action，但会给 Continuation Runtime 提供信号，例如：

```yaml
apply_result:
  outcome: applied
  state_version: 18
  effects:
    opened_questions: []
    closed_questions: [Q-014]
    committed_decisions: []
    pending_human_gates: [HG-004]
    frontier_changed: true
```

Continuation Runtime 可以据此决定：

```text
CONTINUE_FOCUSED
ASK_HUMAN
COMPACT
ROLLOVER
COMPLETE
```

这保持“State transition”和“Next action policy”分离。

---

# 18. 与 Execution Journal 的接口

一次 State Delta apply 是本地 state mutation，不等于 provider action lifecycle。

Journal 后续至少要能引用：

```yaml
action_id:
proposal_id:
delta_id:
apply_result_ref:
```

但不要把：

```text
prompt sent
assistant observed
state proposal accepted
```

压成同一个状态字段。

详细语义留给《Execution Journal & Recovery Contract v0》。

---

# 19. v0 不做什么

明确不做：

- CRDT / distributed merge；
- 自动 stale delta merge；
- 任意 JSON Patch；
- 通用知识图谱；
- source truth 自动刷新引擎；
- reviewer consensus 算法；
- automatic fact adjudication；
- 跨 Run transaction；
- partial apply。

v0 的目标只有：

> **让单 Run 的结构化 State 可以被安全、可追溯、可授权、可测试地演进。**

---

# 20. 建议的实现接口形态

不是最终语言/API，只表达模块边界：

```text
authorize(proposal, state, policy) -> AuthorizationResult
reduce(state, authorized_delta) -> ApplyResult + NewState
audit(proposal, authorization, apply_result) -> AuditRecord
```

`reduce` 应尽量纯函数化：

```text
同样 State + 同样 Authorized Delta
→ 同样结果
```

这样后续可以做 replay、property test 和 crash recovery verification。

---

# 21. 最小测试集

State Delta Contract 在进入实现前至少应有这些 test case：

1. agent 可添加 hypothesis；
2. agent 无权直接改 scope；
3. hypothesis 不能直接变 committed decision；
4. committed decision 不能被 delete；
5. rejected item 不能无 reopen operation 复活；
6. stale base version 被拒绝；
7. precondition 失败不改 State；
8. multi-op delta 任一失败则全部回滚；
9. no-op 不 bump state version；
10. successful delta 只 bump 一次 version；
11. factual claim provenance 可追到 Source Ref；
12. normative claim 不错误服从 agent inference；
13. reviewer stale snapshot 不直接污染当前 State；
14. identical reducer input 可 deterministic replay。

---

# 22. 下一步

这篇 Contract 完成后，下一篇应进入：

> **《Continuation State Machine v0》**

它要回答：

- 什么时候继续；
- 什么时候 focused continue；
- 什么时候 Human Gate；
- completion 如何判定；
- compaction / rollover / refresh 如何成为 Action，而不是顶层业务状态；
- user activity 如何抢占 automation；
- integrity failure 如何进入 reconcile / pause；
- State Delta apply result 如何影响 next action。

Execution Journal 的 dispatch/reconciliation/state-application 多轴状态，则留到第四篇独立 Contract。
