# NOOS Harness｜Continuation State Machine v0

> **状态**：Design Candidate v0.1  
> **日期**：2026-08-27  
> **上位文档**：[NOOS Harness Overview v0.2](overview.md)  
> **依赖基线**：[Runtime Object Model & Authority Model v0](runtime-object-authority-model.md) · [State Delta + Reducer Contract v0](state-delta-reducer-contract.md)  
> **目标**：定义一个 Logical Thread 在最新结果与状态 settle 后，如何安全地继续、暂停、请求人类、维护 context/browser、完成或进入 blocked/reconciliation。

---

## 0. 一句话说明

Continuation Runtime 不是“回答结束以后自动发送继续”。

它负责的是：

> **在 State、Authority 和 Runtime observation 已经达到可判断状态后，决定下一步允许发生什么，并把有副作用或必须持久化的决定变成可恢复、可审计、不会重复创建的 Action Intent。**

核心闭环：

```text
Provider Turn observed
        ↓
Result Ingestion
        ↓
State Proposal(s) / Human Intervention Request(s)
        ↓
Policy / Reducer settle
        ↓
Applied Run State
+ Authorization / Gate state
+ Runtime Signals
+ optional Control Proposal
        ↓
Continuation Evaluation
        ↓
├─ no action / keep waiting
└─ Durable ContinuationDecision
        ↓
Execution / Session Runtime
        ↓
Provider / Browser side effect
        ↓
Observed Runtime Event
        ↓
next cycle
```

Continuation 不替代 Authority Policy，不替代 Reducer，也不负责证明 browser click / provider send 是否真的发生；后者属于 `Execution Journal & Recovery Contract`。

---

# 1. Ownership：Continuation 以 Logical Thread 为控制单元

v0 以 **Logical Thread** 为控制单元，而不是整个 Run。

```text
Run
├─ Main Design
├─ Domain Review
└─ Production Review
```

不同 Thread 未来可以处于不同状态：

```text
Main Design       → RUNNING
Domain Review     → COMPLETED
Production Review → PAUSED_HUMAN
```

因此：

> **一个 Logical Thread 拥有一个 Continuation Controller；Run 可以拥有多个 Controller。**

M0–M2 可以只实现 primary/main thread。Run-level fan-out / reviewer scheduling 属于后续 Workflow Orchestration。

---

# 2. 不使用一个巨大的单轴 state enum

Continuation 同时有两类问题：

1. **控制权是否允许自动推进？**
2. **当前执行循环进行到哪里？**

v0 拆成两个正交维度：

```text
Control Mode
+
Execution Phase
```

---

# 3. Control Mode v0

```text
RUNNING
PAUSED_USER
PAUSED_HUMAN
BLOCKED
COMPLETED
```

其中 `BLOCKED` 必须带 `block_reason`，至少允许：

```text
reconciliation_required
state_integrity_failure
control_basis_unavailable
```

这样不把所有系统问题都错误叫作 reconciliation，也避免为每一种 blocker 扩一个顶层 mode。

## 3.1 RUNNING

满足 guard 时允许 Continuation Policy 产生新的 autonomous Action Intent。

## 3.2 PAUSED_USER

只表示**用户已经明确接管/暂停该 Thread**，而不是“检测到用户刚刚敲了一个键”。

典型来源：

- explicit Pause；
- manual takeover mode；
- user policy 要求手工确认下一轮。

## 3.3 PAUSED_HUMAN

存在 durable PendingHumanGate，当前 Thread 必须等待人类 authority / choice / input。

## 3.4 BLOCKED

系统当前没有安全自主路径，且 blocker 不是正常的 Human Authority Gate。

例如：

- `reconciliation_required`：不知道刚才 external action 是否发生；
- `state_integrity_failure`：关键对象引用断裂；
- `control_basis_unavailable`：没有足够依据可靠产生 next action，且 fallback evaluator 也失败。

原则：

> **执行历史不确定时不重复 side effect；控制依据不充分时不靠裸“继续”掩盖失败。**

## 3.5 COMPLETED

该 Logical Thread 已完成，不再自动产生新工作 Action。

Primary Thread 完成不自动等于整个 Run 完成。

---

# 4. User Activity 是实时 Guard，不等于 Control Mode

这是 v0 的重要修正。

用户可能只是：

- 开始输入；
- 编辑草稿；
- 又删除草稿；
- 手工发送一条补充消息；

这些都不应该自动把 durable controller state 永久改成 `PAUSED_USER`。

因此分开：

```text
user_activity_signal
= transient runtime guard

PAUSED_USER
= durable user control choice
```

## 4.1 用户正在输入

```text
user_activity_signal = active
→ suppress new autonomous dispatch
```

但不自动把 mode 持久化为 PAUSED_USER。

## 4.2 用户明确 Pause / Take Over

```text
→ mode = PAUSED_USER
```

## 4.3 用户手工提交一个 Turn

该 Turn 按正常 Provider Turn 流程 ingest。

处理完成后是否自动恢复 autonomous mode，由 `autonomy_policy` 决定，而不是因为“用户曾输入过”永久暂停。

---

# 5. Execution Phase v0

```text
READY
DISPATCH_PENDING
AWAITING_PROVIDER
INGESTING_RESULT
SETTLING_STATE
MAINTENANCE
```

## 5.1 READY

当前语义状态已经稳定，可以做下一次 Continuation Evaluation。

至少意味着：

- 最新 assistant/user turn 已持久化；
- 与最新 turn 相关的 State Proposal / InterventionRequest 已产生或明确不存在；
- Authorization / ApplyResult 已 settle 到当前可知状态；
- State Store 已消除 local apply ambiguity；
- 没有已知 external dispatch uncertainty。

## 5.2 DISPATCH_PENDING

已经有 durable ContinuationDecision，需要执行 external/browser/runtime action，但 Execution subsystem 尚未证明 side effect 已发生。

```text
Decision created
≠
Action dispatched
```

## 5.3 AWAITING_PROVIDER

Execution subsystem 已有足够事实证明工作 prompt/request 已经 dispatch，正在等待 Provider response。

此 phase **禁止再发第二条 autonomous Continue**。

## 5.4 INGESTING_RESULT

新 Turn 已观察到，正在 capture / fingerprint / parse / extract。

## 5.5 SETTLING_STATE

与最新结果相关的 Proposal / Authorization / Reducer 正在 settle。

Continuation 不能拿旧 State 给最新回答做 next-action 决策。

## 5.6 MAINTENANCE

正在执行不会直接产生新的业务推理结果的 runtime maintenance：

- Safe Refresh；
- reattach；
- same-conversation Compaction；
- Controlled Rollover saga。

具体 maintenance saga 的 step-level recovery 由 Execution Journal 负责。

---

# 6. Continuation Controller 最小持久状态

```yaml
continuation_state:
  run_id:
  logical_thread_id:

  mode: RUNNING
  phase: READY
  block_reason:

  current_decision_id:
  pending_human_gate_id:

  last_state_version_seen:
  last_observed_turn_ref:
  expected_provider_conversation_id:

  pause_reason:
  updated_at:
```

这里不复制整个 Run State，也不复制 Execution Journal。

`expected_provider_conversation_id` 是 guard；真正 current carrier ownership 仍属于 Logical Thread。

---

# 7. ContinuationDecision：durable、immutable、幂等创建

Continuation 的“下一步决定”不能只是内存中的 if/else。

但仅仅 durable 还不够：如果 controller 在“已经持久化 Decision、但调用者没拿到结果”后 crash，重启时也不能基于同一输入再创建一个新的 Decision ID。

因此 v0 定义：

```yaml
continuation_decision:
  id: CD-00042
  run_id:
  logical_thread_id:

  basis:
    state_version:
    provider_conversation_id:
    last_turn_ref:
    latest_gate_refs: []
    authorization_result_ids: []
    apply_result_ids: []
    runtime_signal_snapshot_ref:
    control_proposal_ref:

  basis_fingerprint: sha256:...

  action: CONTINUE_FOCUSED
  next_intent:
  reason_codes: []

  created_at:
```

## 7.1 Basis fingerprint

`basis_fingerprint` 对所有影响决策的**持久输入快照**做指纹。

v0 invariant：

> **同一个 Logical Thread，对同一个 `basis_fingerprint` 至多存在一个有效 ContinuationDecision。**

重启后若发现：

```text
same logical_thread_id + same basis_fingerprint
```

已经有 Decision：

```text
→ reuse existing Decision
```

而不是重新跑策略产生第二个 ID。

若策略需要模型 evaluator，evaluator 的结构化输出必须先成为 durable `ControlProposal` / evaluation artifact，再进入 basis；不能把一次不可复现的临时模型调用藏在 decision transaction 内。

## 7.2 Decision 与 Controller phase 的 local crash consistency

对于需要 execution 的 action，至少应在一个本地 durable transaction 中：

```text
write ContinuationDecision
+
set continuation_state.current_decision_id
+
phase = DISPATCH_PENDING
```

避免出现：

```text
Decision 已 durable
但 controller 仍显示 READY
→ restart 后重复创建 Decision
```

具体 DB 实现不在本文冻结，但 crash-consistency invariant 在本文冻结。

## 7.3 Decision ≠ Execution Fact

> **ContinuationDecision 表示“应该做什么”；Execution Journal 表示“后来实际发生了什么”。**

后续 Execution Journal 应使用 `continuation_decision.id` 作为 correlation / idempotency key。

---

# 8. WAIT 不是 durable Action

旧设计把 `WAIT` 放进 Action Vocabulary，会产生大量没有价值的 durable Decision：

```text
Provider generating
→ WAIT
→ poll
→ WAIT
→ poll
→ WAIT
```

v0 改成：

> **如果当前没有安全可执行动作，Continuation Evaluation 可以返回 `no_action`，不创建 ContinuationDecision。**

例如：

```text
phase = AWAITING_PROVIDER
→ keep waiting
```

```text
user_activity_signal = active
→ suppress dispatch / no new Decision
```

只有需要真正执行一个动作或持久改变控制状态时，才创建 ContinuationDecision。

---

# 9. Action Vocabulary v0

```text
CONTINUE_FOCUSED
COMPACT_CONTEXT
REFRESH_SURFACE
START_ROLLOVER
PAUSE_FOR_HUMAN
PAUSE_FOR_USER
ENTER_BLOCKED
REQUEST_COMPLETE
```

## 9.1 CONTINUE_FOCUSED

不是裸“继续”，而是提供明确 `next_intent`。

最终 prompt 由 Context Compiler / Provider Adapter 构造，Decision 不保存完整大 prompt。

## 9.2 COMPACT_CONTEXT

只表示：

> **在当前 Provider Conversation 内做一次 Stateful Compaction / Carry Context refresh，并在 State settle 后重新 evaluate。**

它不会隐式自动接着执行 Rollover。

如果 compaction 之后仍应 rollover，必须基于**新的 State / health basis** 创建新的 `START_ROLLOVER` Decision。

## 9.3 REFRESH_SURFACE

重建 Browser Session / Adapter Attachment，不更换 Provider Conversation。

## 9.4 START_ROLLOVER

启动 Controlled Context Reset。

v0 中 `START_ROLLOVER` saga **自己包含为了安全 rollover 所必需的 final compaction/checkpoint/projection**；不要要求调用方先发一笔 `COMPACT_CONTEXT` 再假设下一笔一定 rollover。

所以：

- `COMPACT_CONTEXT` = 留在当前 Conversation；
- `START_ROLLOVER` = 进入完整 rollover saga，内部包含必要 compaction。

## 9.5 PAUSE_FOR_HUMAN

切到 PAUSED_HUMAN，并引用已有 durable PendingHumanGate。

## 9.6 PAUSE_FOR_USER

只用于用户明确 Pause/Takeover 或 policy 要求手工模式。

## 9.7 ENTER_BLOCKED

进入 `BLOCKED`，必须有明确 `block_reason`。

## 9.8 REQUEST_COMPLETE

请求 completion governance；不直接改 State。

---

# 10. Continuation Policy 的输入

至少综合四类输入。

## 10.1 Semantic / State

```text
Applied Run State
Committed / Working State
Frontier
Open Questions
Run / Thread status
```

## 10.2 Authority / Human

```text
AuthorizationResult(s)
PendingHumanGate(s)
HumanGateResolution(s)
ApplyResult(s)
```

## 10.3 Runtime

```text
user activity
provider generation state
adapter attachment state
page health
context health
execution ambiguity / integrity signals
```

## 10.4 Control Proposal（可选）

Worker / Shadow Controller 可以产生 advisory proposal：

```yaml
control_proposal:
  id:
  based_on_state_version:
  last_turn_ref:

  progress:
    advanced | low_progress | blocked | candidate_complete

  next_intent:
  completion_candidate:
  blocker_summary:
  state_proposal_refs: []

  created_at:
```

Control Proposal 必须是 advisory，不是 authority。

---

# 11. Human Gate：Authorization Gate 与 Pre-State Intervention Gate

State Delta Baseline 已处理：

```text
Concrete State Proposal
→ Policy
→ requires_human
→ PendingHumanGate
```

但复杂设计中还有：

> **系统知道必须让用户选择，但用户选择前没有唯一 desired state transition。**

例如 A/B 都合理，取决于产品偏好。

这时不能伪造 `commit_decision(A)` 让用户“批准”。

因此 Continuation Candidate 引入：

```yaml
intervention_request:
  id:
  run_id:
  logical_thread_id:

  kind:
    product_choice | missing_requirement | authority_input

  prompt:
  options: []
  basis_refs: []
  request_fingerprint:
  created_at:
```

InterventionRequest 是 durable、immutable 的“需要人类输入”请求，不是 State mutation。

PendingHumanGate 允许两种 origin：

```yaml
pending_human_gate:
  id:
  run_id:
  logical_thread_id:

  origin_kind:
    authorization_result | intervention_request
  origin_ref_id:

  gate_kind:
  prompt:
  status: pending | resolved | cancelled
  created_at:
  resolved_at:
```

---

# 12. HumanGateResolution：用户回答也必须 durable

如果 A/B Gate 的用户回答只存在于 UI callback 内：

```text
user chooses B
↓
Hub crash before new State Proposal
```

答案就会丢。

因此 v0 Candidate 定义 durable immutable resolution record：

```yaml
human_gate_resolution:
  id:
  gate_id:
  run_id:
  logical_thread_id:

  actor_ref:
  resolution_kind:
    approved | rejected | answered | cancelled

  answer_payload:
  answer_fingerprint:
  observed_at:
```

关系：

```text
Authorization Gate
→ approved/rejected
→ Policy re-evaluates original immutable Proposal
```

```text
Intervention Gate
→ answered
→ durable HumanGateResolution
→ derive new State Proposal from answer
→ Policy + Reducer
```

Gate status 与 Resolution 必须 crash-consistent：不能出现“Gate 已 resolved，但找不到回答内容”。

这部分最终是否提升进 Runtime Object Baseline，待本篇接口 Review 后决定。

---

# 13. Deterministic Decision Priority v0

v0 不用黑箱加权分数做顶层 arbitration。

优先级：

```text
0. Explicit User Control
1. Integrity / Execution Ambiguity
2. Existing Human Gate
3. Provider In-flight
4. Completion Already Applied
5. Completion Candidate
6. Maintenance Need (Page × Context jointly)
7. Semantic Progress
```

## 13.1 Explicit User Control

Explicit Pause / Takeover：

```text
→ PAUSED_USER
```

Transient typing 只作为 dispatch guard，不自动持久 Pause。

## 13.2 Integrity / Execution Ambiguity

```text
conversation mismatch
unknown dispatch outcome
state/reference inconsistency
message fingerprint mismatch
```

→ `ENTER_BLOCKED(reconciliation_required | state_integrity_failure)`。

## 13.3 Existing Human Gate

存在 pending Gate：

```text
→ PAUSE_FOR_HUMAN
```

## 13.4 Provider In-flight

```text
→ no_action / keep AWAITING_PROVIDER
```

绝不发送第二条自主消息。

## 13.5 Completion Already Applied

Thread/Run 已正式 completed：

```text
→ COMPLETED
```

## 13.6 Completion Candidate

若最新 settled state 已满足 completion candidate，应在做昂贵 Refresh/Rollover 之前先进入 completion governance：

```text
→ REQUEST_COMPLETE
```

避免“任务其实已经完成，却先为了优化长聊天去 refresh/rollover”。

## 13.7 Maintenance Need：Page × Context 联合判断

Page Health 与 Context Health 是两个不同信号，但**不能写成两个彼此覆盖的顺序优先级**。

使用二维决策：

```text
Context healthy + Page degraded
→ REFRESH_SURFACE
```

```text
Context pressured
→ prefer START_ROLLOVER when safe
```

```text
Context healthy + only semantic carry-context needs cleanup
→ COMPACT_CONTEXT
```

```text
Page degraded + Context pressured
→ START_ROLLOVER when safe
```

只有 Context 仍健康时，才优先用 Refresh 治疗纯 browser lifecycle 问题。

## 13.8 Semantic Progress

```text
advanced
→ CONTINUE_FOCUSED
```

```text
low_progress
→ focused recovery intent
```

```text
blocked by genuine human-only choice
→ InterventionRequest + Gate
```

原则：

> **信息不足、需要多推一轮、局部矛盾，不自动等于 Human Gate。**

---

# 14. User Preemption

User activity 是 dispatch-time guard，必须在**真正 external side effect 发生前最后检查一次**。

## 14.1 Decision 尚未 dispatch

```text
phase = DISPATCH_PENDING
current autonomous Decision exists
↓
user starts typing
```

Execution subsystem 必须 suppress/cancel 这次 autonomous dispatch。

Decision 保留 audit；具体 `suppressed_before_dispatch` 事件留给 Execution Journal。

只有 explicit takeover 才把 mode 改成 PAUSED_USER。

## 14.2 Dispatch outcome 不确定

如果 user takeover 与 dispatch race 导致“不知道 autonomous prompt 是否已经发出”：

```text
→ BLOCKED(reconciliation_required)
```

不能再发一次猜测。

## 14.3 Provider 已 generating

用户 Pause 只阻止下一步 autonomous dispatch；当前 response 仍可以观察、ingest、settle。

---

# 15. Result Ingestion 顺序固定

```text
Turn observed
↓
Persist Turn + fingerprint
↓
INGESTING_RESULT
↓
Parse / obtain Control Proposal（if configured）
↓
Create durable State Proposal(s) / InterventionRequest(s)
↓
SETTLING_STATE
↓
Policy / Reducer settle
↓
State Store resolves local Delta replay vs true stale
↓
Read latest Applied State
↓
Read PendingHumanGate / Gate Resolution
↓
READY
↓
Continuation Evaluation
```

> **Continuation 基于治理完成后的最新状态，而不是看到自然语言最后一句就自动继续。**

---

# 16. Control Proposal 缺失/损坏 ≠ Reconciliation

如果 assistant response 正常，但 bootstrap Control Proposal marker 缺失或 malformed：

这不代表“external execution history 不确定”。

因此不应默认进入 `reconciliation_required`。

推荐：

```text
1. persist full Turn
2. deterministic format repair（仅格式层）
3. fallback local/independent evaluator（若配置）
4. 若仍无可靠 next-action basis
   → BLOCKED(control_basis_unavailable)
```

只有同时发生 dispatch/head/fingerprint 等执行事实不一致时，才使用 `reconciliation_required`。

---

# 17. Safe Refresh Contract

`REFRESH_SURFACE` 只能在 Safe Refresh Window 执行：

```text
Provider not generating
AND no unresolved dispatch uncertainty
AND latest observed turn persisted/fingerprinted
AND current Run/Continuation state durable
AND no unsent user draft
AND current Provider Conversation ref known
```

生命周期：

```text
READY
↓ Decision(REFRESH_SURFACE)
DISPATCH_PENDING / MAINTENANCE
↓
Browser reload
↓
Adapter reattach
↓
resolve Provider Conversation
↓
verify expected head
├─ match → READY
└─ mismatch / unknown → BLOCKED(reconciliation_required)
```

Invariant：

> **Refresh 可以销毁 Browser Session，但不能改变 Logical Thread current Provider Conversation。**

---

# 18. Controlled Rollover Contract

Rollover 是 maintenance saga：

```text
Current Conversation A
      ↓
settle latest State
      ↓
final Stateful Compaction
      ↓
Checkpoint at state vN
      ↓
Context Compiler projection
      ↓
Create Provider Conversation B
      ↓
Inject projection / bootstrap
      ↓
Observe & identify B
      ↓
Attach B
      ↓
Atomically/auditably switch LogicalThread.current A → B
      ↓
mark A rolled_over
      ↓
READY
```

关键 invariant：

1. A 不删除；
2. B 被可靠创建/识别以前，A 仍是 current；
3. v0 一个 Logical Thread 同时至多一个 current carrier；
4. projection 来自 Applied State / immutable evidence refs；
5. saga failure 不能产生两个 current；
6. switch 事实不确定时进入 `BLOCKED(reconciliation_required)`。

Rollover step-level crash recovery 留给 Execution Journal Contract。

---

# 19. Page Health 与 Context Health

## Page Health

回答 browser surface 是否健康。

候选信号：interaction latency、long task、DOM/message growth、scroll/input delay、observer pressure、refresh 后改善幅度。

## Context Health

回答当前 Provider Conversation 是否还是好的 explicit working boundary。

候选信号：semantic phase boundary、repeated reopening、terminology drift、progress density、projection budget pressure、manual rollover request、round/message count。

具体阈值/权重不在 v0 冻结，遵循：

> **instrument first, optimize second.**

---

# 20. Low Progress Policy

`low_progress` 不直接 Human Gate。

先至少执行 focused recovery：

```text
不要扩写已有解释；
指出最关键 remaining unknown；
尝试关闭一个具体 Open Question；
若无法推进，分类 blocker：
- authority choice
- missing evidence
- scope mismatch
- actual completion
- control/system failure
```

之后：

- authority choice → Intervention Gate；
- missing evidence 且工具可取得 → 自主取证；
- scope mismatch → concrete scope Proposal / Gate；
- actual completion → REQUEST_COMPLETE；
- control failure → BLOCKED(control_basis_unavailable)；
- execution ambiguity → BLOCKED(reconciliation_required)。

连续多少轮 low progress 才触发 recovery，不在 v0 冻结，用 Eval 调。

---

# 21. Completion Contract

不接受：

```text
Assistant: “已经完整了。”
→ magically completed
```

至少：

```text
completion_candidate
→ Completion Policy
→ completion State Proposal
→ Policy / Reducer
→ ApplyResult
→ COMPLETED
```

Completion Policy 可检查：

- deliverable；
- blocking Open Questions；
- PendingHumanGate；
- unresolved blocker；
- explicit completion criteria；
- 是否需要 human confirmation。

只有正式 completion transition applied 后，mode 才进入 COMPLETED。

---

# 22. 状态转换主路径

## 22.1 正常自主循环

```text
mode=RUNNING
phase=READY
↓ evaluate
Durable Decision(CONTINUE_FOCUSED)
+ current_decision_id + DISPATCH_PENDING atomic local persist
↓ dispatch proven
AWAITING_PROVIDER
↓ Turn observed
INGESTING_RESULT
↓
SETTLING_STATE
↓ settled
READY
```

## 22.2 No-action waiting

```text
AWAITING_PROVIDER
↓ evaluate/runtime tick
no_action
↓
remain AWAITING_PROVIDER
```

不创建 WAIT Decision。

## 22.3 Human Gate

```text
SETTLING_STATE / READY
↓ PendingHumanGate exists
PAUSED_HUMAN
↓ durable HumanGateResolution
↓ Policy / Proposal / State settle
↓ no other gate
RUNNING + READY（若 auto-resume policy 允许）
```

## 22.4 Explicit User Pause

```text
any phase
↓ explicit pause/takeover
PAUSED_USER
```

Transient typing 只 suppress dispatch。

## 22.5 Refresh

```text
RUNNING + READY
↓ page degraded + context healthy + safe window
Decision(REFRESH_SURFACE)
↓
MAINTENANCE
↓ reattach + head verified
READY
```

## 22.6 Rollover

```text
RUNNING + READY
↓ context pressure + safe window
Decision(START_ROLLOVER)
↓
MAINTENANCE saga
↓ switch verified
READY
```

## 22.7 Blocked

```text
any phase
↓ integrity / execution / control basis failure
BLOCKED(reason)
```

Recovery / user action / evaluator 修复后才能回 RUNNING。

---

# 23. Autonomy Policy

同一 State Machine 支持不同自动化强度：

```yaml
autonomy_policy:
  mode:
    manual | supervised | autonomous

  auto_resume_after_human_gate: false
  auto_resume_after_manual_turn: true

  allowed_actions:
    - CONTINUE_FOCUSED
    - COMPACT_CONTEXT
    - REFRESH_SURFACE
    - START_ROLLOVER

  unattended_budget:
    max_steps:
    max_wall_time:
```

默认数值不冻结。

- manual：观察/维护，不自动发工作 prompt；
- supervised：产生 Action Intent，但 dispatch 要用户确认；
- autonomous：在 authority/policy 内自动推进。

---

# 24. 与 State Delta 的接口

Continuation 不直接改 Run State。

读取：

```text
latest Applied Run State
AuthorizationResult(s)
PendingHumanGate(s)
HumanGateResolution(s)
ApplyResult(s)
```

产生 state change 时：

```text
Continuation / Worker
→ durable immutable Proposal
→ Policy
→ Reducer / State Store
```

特别是：

```text
REQUEST_COMPLETE
≠ direct set completed
```

```text
scope blocker
≠ direct edit scope
```

```text
Intervention answer
→ durable HumanGateResolution
→ new Proposal
```

State Store 必须先通过 `delta_id` idempotency 消除“已经成功但 receipt 丢失”，Continuation 才能把 `rejected_stale` 当成真正 stale。

---

# 25. 与 Execution Journal 的边界

Continuation 定义：

> **应该做什么，以及 Controller 当前允许什么。**

Execution Journal 定义：

> **这个 Decision 实际 dispatch 了吗、Provider/Browser side effect 是否发生、response 是哪一个、maintenance saga 到哪一步、crash 后怎么 reconcile。**

```text
ContinuationDecision
        ↓
Execution Journal / Dispatcher
        ↓
Observed Runtime Event
        ↓
Continuation State transition
```

`DISPATCH_PENDING → AWAITING_PROVIDER` 必须由 Journal 中的事实驱动，而不是 UI 猜测。

---

# 26. v0 必须测试的场景

### A. Normal Auto Continue

READY → durable Decision → dispatch → response → settle → READY；不可 duplicate send。

### B. Same-basis Decision replay

```text
Decision CD-42 已持久化
process crash before caller receives it
↓ restart
same basis_fingerprint
→ reuse CD-42
→ do not create CD-43
```

### C. Provider still generating

→ no_action；不创建 WAIT Decision。

### D. User typing before dispatch

→ transient guard suppresses autonomous dispatch；不自动永久 PAUSED_USER。

### E. Explicit user takeover

→ PAUSED_USER。

### F. Concrete Proposal needs Human

AuthorizationResult.requires_human → Gate → PAUSED_HUMAN。

### G. A/B preference before unique mutation

InterventionRequest → Gate → durable HumanGateResolution → new Proposal。

### H. Crash after user answer before Proposal generation

HumanGateResolution 仍存在，可恢复生成 Proposal；不能要求用户重新回答。

### I. Page degraded + Context healthy

→ REFRESH_SURFACE。

### J. Page degraded + Context pressured

→ START_ROLLOVER when safe；不能因固定 priority 先做无意义 Refresh。

### K. Completion candidate + Page degraded

→ completion governance first；不先为了性能维护而刷新已接近完成的 Thread。

### L. Refresh head mismatch

→ BLOCKED(reconciliation_required)。

### M. Control Proposal malformed but execution facts consistent

→ fallback evaluator；失败则 BLOCKED(control_basis_unavailable)，不是 reconciliation。

### N. Rollover B creation fails

A 保持 current；不能出现两个 current carrier。

### O. Low progress

focused recovery before Human Gate。

### P. Local State Delta replay

State Store 返回原 ApplyResult；Continuation 不把它误判为 stale/reconciliation。

### Q. User/dispatch race

若无法证明 autonomous prompt 是否已经发出 → BLOCKED(reconciliation_required)。

---

# 27. 当前冻结候选结论

1. **Continuation Controller 以 Logical Thread 为控制单元。**
2. **Control Mode 与 Execution Phase 正交。**
3. **Transient user activity 是 runtime guard；只有 explicit pause/takeover 才进入 durable PAUSED_USER。**
4. **ContinuationDecision durable + immutable，并按 basis_fingerprint 幂等创建。**
5. **同一 Logical Thread / 同一 basis 至多一个有效 Decision。**
6. **Decision 创建与 current_decision_id / DISPATCH_PENDING 需要 local crash consistency。**
7. **WAIT 是 no-action outcome，不创建 durable Decision。**
8. **Provider in-flight 时绝不自主发送第二条消息。**
9. **Human Gate 有 Authorization 与 Intervention 两类 origin。**
10. **HumanGateResolution durable，用户答案不能在 crash 中丢失。**
11. **Continuation 不直接改 Run State，所有 mutation 继续走 Proposal → Policy → Reducer。**
12. **COMPACT_CONTEXT 与 START_ROLLOVER 不隐式串联；Rollover saga 自带必要 final compaction/checkpoint。**
13. **Page Health × Context Health 联合决策，而不是两个互相打架的顺序优先级。**
14. **Completion candidate 优先于非必要 Refresh/Rollover。**
15. **Control Proposal 失败与 execution reconciliation 是不同 blocker。**
16. **Low Progress 先 focused recovery，不直接 Human Gate。**
17. **State Store local apply idempotency 在 Continuation 之前消除 replay ambiguity。**
18. **Execution Journal 负责“实际发生什么”，Continuation 负责“下一步应该做什么”。**

---

# 28. 下一步接口 Review

本文继续保持 **Design Candidate v0.1**。

下一步不直接进入 Recovery Contract，而先做一次：

> **Runtime Object Model × State Delta × Continuation 三篇接口对审。**

重点只审新出现/被强化的接口：

```text
InterventionRequest
HumanGateResolution
ContinuationState
ContinuationDecision
basis_fingerprint / single active decision
user-preemption guard
State Apply idempotency → true stale
```

对审通过后，再决定：

1. 哪些对象需要提升进 Runtime Object Baseline；
2. Continuation 是否可以升 Design Baseline；
3. 然后进入 Execution Journal & Recovery Contract v0。
