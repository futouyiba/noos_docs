# NOOS Harness｜State Delta + Reducer Contract v0

> **状态**：Design Candidate v0  
> **日期**：2026-08-27  
> **上位文档**：[Runtime Object Model & Authority Model v0](runtime-object-authority-model.md)  
> **目标**：定义 Proposal 如何经过 durable Authorization，在事务边界内安全、可追溯、可并发检测地变成 Run State 变更。

---

## 0. 一句话说明

Harness 不能让 LLM 每隔几轮“重写一份最新 State”。

v0 的状态变化管线冻结为：

```text
Current State vN
      +
Atomic Proposal
      ↓
Authority / Promotion Policy
      ↓
Durable AuthorizationResult
      ├─ requires_human → PendingHumanGate
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

Proposal 是 Worker / Reviewer / User / Source Observer 提出的**候选状态事务请求**。

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

Proposal 不代表 State 已改变，也不代表 proposer 拥有执行这些 operation 的 authority。

最重要的语义是：

> **Proposal 本身就是“请求”；Operation 描述期望发生的最终 State Transition。**

因此 v0 不再使用 `propose_decision`、`propose_scope_change` 这类二次“提议操作”。

Agent 完全可以提交：

```yaml
operations:
  - op: commit_decision
```

这只表示“请求 committed state 出现这个 decision”，并不表示 Agent 有权 commit。Authority Policy 决定它是否成立。

## 2.2 AuthorizationResult

Policy 对**整个 Proposal 事务**给出 durable 结果：

```yaml
authorization_result:
  authorization_result_id: AZ-000087
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

AuthorizationResult 必须持久化。它是 Policy 的正式输出，不是临时函数返回值。

## 2.3 Authorized Delta

只有 `AuthorizationResult.status = authorized` 时，Policy 才产生 Authorized Delta：

```yaml
authorized_delta:
  delta_id: SD-000087
  proposal_id: P-000123
  authorization_result_id: AZ-000087
  run_id: run-abc
  base_state_version: 17
  operations: []
```

Authorized Delta 在 v0 中与 Proposal 具有相同事务边界，不允许 Policy 从一个 Proposal 中只摘取部分 operation 继续执行。

## 2.4 ApplyResult

Reducer 处理 Authorized Delta 后返回：

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

ApplyResult 不创建 Human Gate。Human Gate 属于 Policy / Authorization 边界。

---

# 3. Proposal 是 v0 最小逻辑事务单元

这是 v0 的关键 invariant。

例如一个逻辑变化：

```text
commit D-023
+
close Q-014 using D-023
+
move frontier to Q-015
```

应该形成**一个 Proposal**。

而两个相互独立的变化：

```text
add hypothesis H-9
```

和：

```text
request a product decision requiring human authority
```

应该形成**两个 Proposal**。

因此 v0 不引入 `atomic_group` / dependency graph / partial authorization。

原则是：

> **Proposal atomicity → Authorization atomicity → Reducer atomicity。**

这使三层事务语义一致。

---

# 4. State 分区与写权限

基于 Runtime Object Model：

```yaml
run_state:
  operational:
  committed:
  working:
  sources:
  meta:
```

## 4.1 Operational

例如：goal、deliverable、scope、phase、status。

`scope` / `goal` 通常需要更高 authority；`phase` 可以由 delegated policy 自动推进。

## 4.2 Committed

```yaml
committed:
  constraints: []
  decisions: []
  rejected: []
```

这些是 Run 内正式提交的治理状态。

任何新增、supersede、reopen 都必须经过 Authority / Promotion Policy。

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

Run State 只保存 SourceRef / EvidenceRef 的引用关系：

```yaml
sources:
  source_ref_ids: []
  evidence_ref_ids: []
```

`SourceRef` 表示“观察了什么”；`EvidenceRef` 表示“这个 Source observation 对哪个 Claim 起什么证据作用”。

详见上位 Object / Authority Model。

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

例如 Agent 提交 `commit_decision` 后，Policy 可能返回：

```text
requires_human
```

而不是让 Reducer直接执行。

## 5.3 Operational operations

```text
set_phase
set_status
commit_scope_change
commit_goal_change
```

同样，`commit_scope_change` 是 desired transition，不代表 proposer 拥有 scope authority。

## 5.4 Source / Evidence operations

v0 只允许显式增加 immutable observation 与 evidence relationship：

```text
attach_source_ref
attach_evidence_ref
record_source_supersession
```

不提供：

```text
update_source_observation
```

因为已经被 provenance 引用的 observation identity 不允许原地改写。

`mark_source_stale` 若未来需要，可以作为 freshness/index 层的派生状态，不应通过修改历史 SourceRef 内容身份实现。

## 5.5 明确禁止的通用操作

```text
delete_anything
replace_state
set_arbitrary_path
raw_json_patch
```

它们会绕过语义 invariant，重新制造“模型重写整个 State”的风险。

---

# 6. Operation Envelope

每个 operation 使用统一 envelope：

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

Provenance 的目的不是复制全部聊天，而是回答：

- 这个 Decision 从哪里来？
- 为什么 Q-014 被关闭？
- Reviewer 基于哪个 snapshot 提出 blocker？
- 这个 factual claim 依据哪个 source observation？

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

```yaml
evidence_ref_id: E-009
```

EvidenceRef 再指向 immutable SourceRef：

```text
Claim / State Entity
      ↓
EvidenceRef
      ↓
SourceRef observation
```

因此历史 provenance 不会因为“重新观察同一文档/代码”而改变。

## 7.3 Review provenance

```yaml
review_issue_ref:
review_snapshot_id:
base_state_version:
```

若 Review Snapshot 已 stale，Policy 可拒绝或要求重新生成 Proposal；Reducer 不猜测如何 rebase。

---

# 8. Source observation immutability

一旦 SourceRef 被 EvidenceRef / Decision provenance 引用，它的内容身份字段不可原地覆盖。

例如：

```text
S-004 = Git commit A, observed_at T1, fingerprint F1
```

后来 Git 已到 commit B：

```text
创建 S-011 = Git commit B, observed_at T2, fingerprint F2
S-011.supersedes_source_ref_id = S-004
```

而不是把 S-004 从 A 改成 B。

这样：

```text
D-017 → E-003 → S-004
```

永远表达“D-017 当时基于什么证据形成”。

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

若内部检查结果存在：

```text
任一 denied
```

→ 整个 Proposal：

```text
status = denied
```

否则若存在：

```text
任一 requires_human
```

→ 整个 Proposal：

```text
status = requires_human
```

只有全部 operation 都可授权：

```text
status = authorized
```

→ 才生成 Authorized Delta。

Policy 可以在 AuthorizationResult 中保留 operation-level diagnostic，用于解释“是哪一项导致整笔事务被 gate/deny”，但不能把其余 operation 偷偷拆出去执行。

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

`close_question` 是否 delegated 取决于 question 类型：纯 exploratory question 可允许；涉及 product choice / scope / committed decision 的问题应 gate。

## 9.3 Claim 类型影响 Authority

- `preference / constraint / decision`：用户或被授权 owner 的明确表达通常高于 agent inference；
- `fact`：依据对应领域 canonical/current EvidenceRef，而不是固定 `user > document`；
- `inference`：默认保持 working hypothesis，除非走 promotion。

---

# 10. Human Gate 的 durable ownership

`requires_human` 发生在 Policy 阶段，不属于 Reducer ApplyResult。

Policy 必须持久化：

```yaml
authorization_result:
  authorization_result_id: AZ-000088
  proposal_id: P-000123
  status: requires_human
  gate_id: HG-004
  authority_basis: product_choice
  reason:
  created_at:
```

以及：

```yaml
pending_human_gate:
  gate_id: HG-004
  run_id: run-abc
  proposal_id: P-000123
  authorization_result_id: AZ-000088
  gate_kind: product_choice
  prompt:
  status: pending | approved | rejected | cancelled
  created_at:
  resolved_at:
```

这样即使 Browser / Hub 在用户回答前崩溃，Run 仍知道有一个未解决的 authority boundary。

## 10.1 Human approval 后如何继续

v0 推荐保留 audit trail：

```text
AZ-000088 = requires_human
HG-004 = approved
        ↓
Policy re-evaluates original Proposal
        ↓
AZ-000091 = authorized
supersedes AZ-000088
        ↓
Authorized Delta
```

不要把旧 AuthorizationResult 原地改成 authorized。

---

# 11. Versioning 与 Optimistic Concurrency

每个 Proposal / Authorized Delta 都携带：

```yaml
base_state_version: N
```

Reducer 只在：

```text
current_state.version == base_state_version
```

时直接 apply。

否则：

```text
rejected_stale
```

v0 **不允许 Reducer 自动 merge stale delta**。

上层可以：

- 重新生成 Proposal；
- 重新 review；
- 让 Policy/Orchestrator显式 rebase；
- 请求人处理。

---

# 12. Preconditions

`base_state_version` 防整体 stale；operation precondition 表达局部语义前提。

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

例如：

```yaml
op: reopen_decision
preconditions:
  - kind: committed_decision_active
    target_id: D-017
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

一个 Proposal：

```text
OP1
OP2
OP3
```

只可能出现：

```text
整个 denied
整个 requires_human
整个 authorized → 整个 apply/no-op/reject
```

不会出现：

```text
OP1 等人
OP2 先写 State
OP3 被拒绝
```

若调用方有独立 mutation，调用方负责拆成多个 Proposal。

未来若需要复杂 batch，再独立引入 `atomic_group` / dependency contract；v0 不做。

---

# 14. Reducer Invariants v0

Reducer 必须是确定性、无模型推理的状态转换器。

## 14.1 不允许静默删除 Committed State

Committed Decision / Constraint / Rejection 不能直接消失。

只能显式：

```text
active → superseded
active → reopened
```

并保留历史与 provenance。

## 14.2 Rejected 不允许直接复活

必须通过 `reopen_rejection`，并引用 reopen reason/evidence。

## 14.3 Working ≠ Committed

`add_hypothesis` 不能产生 committed decision。

若需要把 Hypothesis 提升为 Decision，Proposal 使用显式 `commit_decision` 并引用 hypothesis/provenance，Policy 决定是否授权。

## 14.4 ID 唯一

所有一等 State entity ID 在 Run 内必须唯一。

## 14.5 引用完整

例如 `close_question.resolution_ref` 必须指向存在且合法的 resolution entity。

## 14.6 Scope / Goal 不能被通用字段写入绕过

不存在 `set_field` 路径可以绕过 `commit_scope_change` / `commit_goal_change` 与 Policy。

## 14.7 Evidence provenance 不可被 observation overwrite

Committed factual claim 若引用 EvidenceRef → SourceRef，重新观察 source 不能修改旧 SourceRef 的内容身份。

## 14.8 Version 单调递增

至少一项实际 mutation 成功时：

```text
vN → vN+1
```

失败或全部 no-op 不产生新 State version。

---

# 15. Reducer 不应该做什么

Reducer **不做**：

- LLM 语义判断；
- 事实搜索；
- Authority / Human Gate 判断；
- 自动裁决 reviewer 冲突；
- 自动 rebase stale Proposal；
- 执行 external side effect；
- 自动决定 next action；
- 创建 PendingHumanGate。

这些分别属于 Policy、Source Resolver、Review Orchestrator、Continuation Runtime、Execution subsystem。

Reducer 应尽量接近一个可单元测试的 pure-ish function。

---

# 16. Conflict / Stale / No-op 语义

## 16.1 Stale

```text
base_state_version != current_state_version
```

→ `rejected_stale`

## 16.2 Precondition failure

State version 没变，但 operation 的局部前提不成立：

→ `rejected_precondition`

## 16.3 Invariant failure

例如引用不存在对象、试图静默删除 committed entity：

→ `rejected_invariant`

## 16.4 No-op

例如 operation 期望的状态已经成立。

规则冻结为：

```text
所有 operation 都是 no-op
→ outcome = no_op
→ state_version 不变
```

```text
至少一项产生实际 mutation
且其余 operation 均成功或 no-op
→ outcome = applied
→ state_version + 1
```

`no-op` 不视为事务失败。

---

# 17. AuthorizationResult 与 ApplyResult 的接口边界

这两个对象不能混。

## AuthorizationResult

Policy owns：

```text
authorized
requires_human
denied
PendingHumanGate creation
```

## ApplyResult

Reducer owns：

```text
applied
no_op
rejected_stale
rejected_precondition
rejected_invariant
```

因此后续 Continuation Runtime 的输入应是：

```text
Run State
+ latest AuthorizationResult / PendingHumanGate
+ ApplyResult（若 Reducer 执行过）
+ Runtime Signals
```

而不是让 Reducer 用 `effects.pending_human_gates` 偷偷承担 Policy 职责。

---

# 18. Review / stale Proposal

Reviewer Proposal 必须携带：

```yaml
review_snapshot_id:
base_state_version:
```

如果主线程已经推进：

```text
base_state_version != current
```

Reducer 直接 `rejected_stale`。

是否重新生成 Reviewer Proposal、是否仍保留原 issue 的价值，由 Review Orchestrator / Policy 决定。

---

# 19. Compaction Worker 的输出

Compaction Worker 不拥有 State。

它的输出仍然是一个或多个**彼此独立、各自 atomic 的 Proposal**：

```text
Transcript Segment
      ↓
State Extraction
      ↓
Proposal(s)
      ↓
Policy
      ↓
Reducer
```

若同一段 Compaction 产生两个互不依赖的变化，可以拆成两个 Proposal；若三个 operation 必须共同成立，则必须放在同一 Proposal。

禁止：

```text
Transcript
→ new_state.yaml
→ overwrite
```

---

# 20. Context Compiler 只消费 Applied State

Context Compiler 可以知道存在 PendingHumanGate / denied Proposal，但不能把它们伪装成 Committed State。

例如：

```text
Committed Decision
→ 可以作为正式约束投影

Pending Proposal requesting Decision
→ 只能标为 Pending / Awaiting Approval
```

这样模型不会因为“自己刚提出一个方案”就在下一轮把它当成已确认事实。

---

# 21. State Delta v0 的最小实现范围

真正 M0/M1 需要实现的 operation 可以进一步收窄：

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
record_source_supersession
```

其他 operation 可以在需要时扩展，不必一次全部完成。

---

# 22. 必须测试的场景

至少覆盖：

### A. Agent 请求普通 Working mutation

```text
Proposal(add_hypothesis)
→ authorized
→ applied
```

### B. Agent 请求 Product Decision

```text
Proposal(commit_decision)
→ requires_human
→ durable PendingHumanGate
→ crash/restart
→ gate 仍存在
→ human approves
→ new AuthorizationResult authorized
→ applied
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

### D. Stale reviewer

```text
Proposal base v17
Current v20
→ rejected_stale
```

### E. Rejected option 无条件复活

```text
reopen_rejection without valid authorized path
→ rejected / invariant failure
```

### F. Source re-observation

```text
S-004 = commit A
new observation = commit B
→ create S-011
→ do not overwrite S-004
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

# 23. 当前冻结结论

1. **Proposal 是 request；Operation 是 desired final state transition。**
2. **v0 不存在 `propose_*` Reducer operation。**
3. **Proposal 是最小逻辑事务单元。**
4. **Authorization 与 Reducer 都遵守同一事务边界。**
5. **AuthorizationResult 必须 durable。**
6. **Human Gate 由 Policy 创建并持久化，不由 Reducer产生。**
7. **Authorized Delta 只在整笔 Proposal authorized 时存在。**
8. **Reducer deterministic，不做 authority/LLM/source reasoning。**
9. **SourceRef observation immutable；新观察创建新 SourceRef。**
10. **EvidenceRef 承载 claim_kind / authority_role；Source 与 Claim 分离。**
11. **stale Proposal 不自动 merge。**
12. **全部 no-op 不 bump version；mixed no-op + mutation 算 applied。**
13. **Context Compiler 只把 Applied/Committed State 当正式工作状态。**

---

# 24. 与下一层 Continuation 的接口

Continuation State Machine 不需要猜 State Delta 内部细节。

它只需要消费清晰的结果对象：

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

这条接口钉死以后，下一篇 Continuation Contract 才有稳定地基。

---

## 当前状态

本文继续保持 **Design Candidate v0**。

本轮已关闭：

- Proposal vs `propose_*` 双重建模；
- per-operation authorization 与 all-or-nothing atomicity 冲突；
- `requires_human` 缺少 durable owner；
- Source observation 可被原地覆盖的问题；
- mixed no-op 语义空洞。

下一步应与 Runtime Object & Authority Model 做一次接口对审；通过后再考虑两篇一起升 Baseline，并进入 Continuation State Machine。
