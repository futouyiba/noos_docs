# NOOS Harness｜Continuation State Machine v0

> **状态**：Design Candidate v0  
> **日期**：2026-08-27  
> **上位文档**：[NOOS Harness Overview v0.2](overview.md)  
> **依赖基线**：[Runtime Object Model & Authority Model v0](runtime-object-authority-model.md) · [State Delta + Reducer Contract v0](state-delta-reducer-contract.md)  
> **目标**：定义一个 Logical Thread 在结果到达以后，如何安全地决定继续、暂停、维护、rollover、完成或进入 reconciliation。

---

## 0. 一句话说明

Continuation Runtime 不是“回答结束以后自动发送继续”。

它真正负责的是：

> **在 Run State 已经稳定、Authority 结果已经明确、当前网页/Provider 状态已经被观察之后，决定下一步允许发生什么，并把这个决定变成可恢复、可审计的 Action Intent。**

核心闭环：

```text
Provider Turn observed
        ↓
Result Ingestion
        ↓
State Proposal(s)
        ↓
Policy / Reducer settle
        ↓
Run State + AuthorizationResult + ApplyResult
        +
Runtime Signals
        +
Control Proposal（可选）
        ↓
Continuation Policy
        ↓
Durable ContinuationDecision
        ↓
Execution / Session Runtime
        ↓
Provider / Browser side effect
        ↓
next observation
```

Continuation 不替代 Authority Policy，不替代 Reducer，也不负责解释一次 browser click 到底有没有成功。后者属于后续 Execution Journal & Recovery Contract。

---

# 1. 先钉死 ownership：Continuation 是谁的状态机？

v0 以 **Logical Thread** 为控制单元，而不是整个 Run。

原因是：

```text
Run
├─ Main Design
├─ Domain Review
└─ Production Review
```

未来三个 Thread 可以处于不同状态：

```text
Main Design       → RUNNING
Domain Review     → COMPLETED
Production Review → PAUSED_HUMAN
```

因此：

> **一个 Logical Thread 拥有一个 Continuation Controller；Run 可以拥有多个 Controller。**

M0–M2 可以只实现 primary/main thread，但对象边界先按这个模型冻结。

Run-level fan-out / reviewer scheduling 属于后续 Workflow Orchestration，不塞进本文。

---

# 2. 不要用一个巨大的单轴 state enum

Continuation 同时存在两类不同问题：

1. **控制权现在是否允许自动推进？**
2. **当前一次执行循环进行到哪一步？**

如果把它们压成：

```text
WAITING
PAUSED
GENERATING
REFRESHING
HUMAN
INGESTING
ROLLOVER
...
```

很快会发生状态爆炸。

v0 因此拆成两个正交维度：

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
BLOCKED_RECONCILIATION
COMPLETED
```

## 3.1 RUNNING

允许 Continuation Policy 在满足 guard 时产生新的 autonomous Action Intent。

## 3.2 PAUSED_USER

用户显式 Pause，或用户正在输入/接管当前 Thread。

原则：

> **用户操作优先于自动化。**

进入 PAUSED_USER 并不意味着丢掉当前 in-flight provider response；已有结果仍可以被观察、持久化和 ingest，只是不再自动发送下一步。

## 3.3 PAUSED_HUMAN

存在一个 durable PendingHumanGate，当前工作必须等待人类 authority / choice / input。

Continuation 不应不断重新问相同问题；它只引用已有 Gate。

## 3.4 BLOCKED_RECONCILIATION

系统无法证明自己知道“刚才到底发生了什么”或发现关键对象不一致，例如：

- 当前网页 Conversation 与 Logical Thread current carrier 不匹配；
- dispatch 是否发生无法确定；
- last observed message fingerprint 不一致；
- State / Proposal / Authorization reference 断裂；
- refresh 后页面位置无法和 journal 对齐。

这种情况不能靠“再点一次 Continue”猜过去。

> **不确定执行历史时，宁可进入 reconciliation，也不要重复 side effect。**

具体如何 reconcile 留给 Execution Journal & Recovery Contract。

## 3.5 COMPLETED

该 Logical Thread 已经完成，不再自动产生新的工作 Action。

Primary Thread 的完成不自动等于整个 Run 完成；Run completion 仍由 Run-level completion policy / State transition 决定。

---

# 4. Execution Phase v0

```text
READY
DISPATCH_PENDING
AWAITING_PROVIDER
INGESTING_RESULT
SETTLING_STATE
MAINTENANCE
```

## 4.1 READY

当前语义状态已经稳定，可以做下一步 Continuation Decision。

“稳定”至少意味着：

- 当前 assistant turn 已经完整观察；
- 与该 turn 相关的 State Proposal 已经产生或明确不存在；
- Authorization / ApplyResult 已经 settle 到当前可知状态；
- 没有已知 dispatch uncertainty。

## 4.2 DISPATCH_PENDING

已经有 durable ContinuationDecision，但对应 external/browser action 尚未被 Execution subsystem 证明已发出。

这个 phase 非常重要：

```text
决定发送
≠
已经发送
```

Execution Journal 后续负责区分 planned / dispatched / observed 等事实。

## 4.3 AWAITING_PROVIDER

Execution subsystem 已有足够证据认为请求已经发送，当前等待 Provider response。

Continuation 在这个 phase 不允许再发第二条 Continue。

## 4.4 INGESTING_RESULT

Provider 新结果已经观察到，正在：

- capture Turn；
- 计算 fingerprint；
- 解析 Control Proposal（若采用 bootstrap marker）；
- 抽取 state-related request。

## 4.5 SETTLING_STATE

与最新 Turn 相关的 Proposal / Authorization / Reducer 正在收敛。

Continuation 必须等这一步结束以后再决定下一步；不能拿旧 State 对最新回答做决策。

## 4.6 MAINTENANCE

正在执行不会产生新业务推理结果的 runtime maintenance，例如：

- Safe Refresh；
- reattach；
- Controlled Rollover saga；
- context compaction / checkpoint preparation。

Maintenance 结束后回到 READY，或因为不一致进入 BLOCKED_RECONCILIATION。

---

# 5. Continuation Controller 最小持久状态

建议：

```yaml
continuation_state:
  run_id:
  logical_thread_id:

  mode: RUNNING
  phase: READY

  current_decision_id:
  pending_human_gate_id:

  last_state_version_seen:
  last_observed_turn_ref:
  expected_provider_conversation_id:

  pause_reason:
  updated_at:
```

这里故意不复制整个 Run State，也不复制 Execution Journal。

`expected_provider_conversation_id` 是 runtime guard，不是第二份 ownership；真正 current carrier 仍属于 Logical Thread。

---

# 6. ContinuationDecision：Decision 必须 durable

与 State Proposal 同样，Continuation 的“下一步决定”不能只是一个内存中的 if/else 返回值。

否则：

```text
Policy 决定发送 Continue
↓
进程崩溃
↓
重启后重新算一次
↓
可能重复发送
```

因此 v0 定义 durable ContinuationDecision：

```yaml
continuation_decision:
  id: CD-00042
  run_id:
  logical_thread_id:

  basis:
    state_version:
    provider_conversation_id:
    last_turn_ref:
    authorization_result_ids: []
    apply_result_ids: []
    runtime_signal_snapshot_ref:

  action: CONTINUE_FOCUSED
  next_intent:
  reason_codes: []

  created_at:
```

原则：

> **ContinuationDecision 表示“系统决定下一步应该做什么”；Execution Journal 表示“这个决定后来实际发生了什么”。**

后续 Execution Journal 应以 `continuation_decision.id` 作为重要 idempotency / correlation key。

---

# 7. Action Vocabulary v0

Continuation 不需要几十种 action。

第一版保持：

```text
CONTINUE_FOCUSED
COMPACT_CONTEXT
REFRESH_SURFACE
START_ROLLOVER
PAUSE_FOR_HUMAN
PAUSE_FOR_USER
ENTER_RECONCILIATION
REQUEST_COMPLETE
WAIT
```

## 7.1 CONTINUE_FOCUSED

不是裸：

> 继续。

而是给 Context Compiler 一个明确的 `next_intent`，例如：

```text
继续处理 Q-014；不要重新总结已关闭内容；
优先检查当前拆分是否产生 double-count，争取关闭一个 open question。
```

最终 prompt 由 Context Compiler / Provider Adapter 构造，ContinuationDecision 不需要保存完整大 prompt。

## 7.2 COMPACT_CONTEXT

触发一次 stateful compaction / carry-context refresh。

它可以独立发生，也可以作为 Rollover 的前置 maintenance。

Compaction 不允许 whole-state rewrite；仍遵守 State Delta Contract。

## 7.3 REFRESH_SURFACE

只重建 Browser Session / Adapter Attachment，不更换 Provider Conversation。

## 7.4 START_ROLLOVER

启动 Controlled Context Reset：旧 Provider Conversation 保留 provenance，新 Conversation 成为同一个 Logical Thread 的新 current carrier。

## 7.5 PAUSE_FOR_HUMAN

Continuation 本身不篡改 State，也不“假装批准”。它将 mode 切到 PAUSED_HUMAN，并引用 durable PendingHumanGate。

## 7.6 PAUSE_FOR_USER

用户主动接管时停止自动下一步。

## 7.7 ENTER_RECONCILIATION

当执行历史/页面/State 之间无法可靠对齐时进入保护状态。

## 7.8 REQUEST_COMPLETE

“模型说做完了”不能直接让 Thread / Run 消失。

Continuation 只能提出 completion request。若 completion 需要修改 Run operational status，应继续走 State Proposal → Policy → Reducer。

只有完成状态真正被应用，或 Thread-level completion policy 明确允许后，mode 才进入 COMPLETED。

## 7.9 WAIT

当前没有安全可执行动作，例如 Provider 仍 generating。

WAIT 不产生 external side effect。

---

# 8. Continuation Policy 的输入

Continuation 不应该只读模型一句“下一步建议”。

至少综合四类输入。

## 8.1 Semantic / State inputs

```text
Run State
Committed State
Working State
Frontier
Open Questions
Run / Thread status
```

## 8.2 Authority inputs

```text
latest AuthorizationResult(s)
PendingHumanGate(s)
latest ApplyResult(s)
```

## 8.3 Runtime inputs

```text
user activity
provider generation state
adapter attachment state
page health
context health
execution ambiguity / reconciliation signal
```

## 8.4 Control Proposal（可选）

Bootstrap 阶段可以让 Worker / Shadow Controller 给一个 advisory proposal：

```yaml
control_proposal:
  based_on_state_version:
  last_turn_ref:

  progress:
    advanced | low_progress | blocked | candidate_complete

  next_intent:
  completion_candidate:
  blocker_summary:
  state_proposal_refs: []
```

Control Proposal **不是 authority**。

> Worker 可以建议“继续 / 完成 / 需要用户”，最终 action 仍由 Continuation Policy 决定。

---

# 9. 一个必要的 Human Gate 泛化：Authorization Gate ≠ 所有 Human Gate

State Delta Baseline 已经正确处理一种 Gate：

```text
Concrete State Proposal
→ Policy
→ requires_human
→ PendingHumanGate
```

例如：

```text
“把 D-021 supersede 为 D-044”
```

这是一个已经明确的 desired transition，只是缺人类 authority。

但复杂设计里还有另一类情况：

> **系统知道必须让用户做选择，但在用户选择以前，还不存在唯一的 desired state transition。**

例如：

```text
方案 A 与 B 都成立；
选择取决于产品偏好。
```

这时不应该伪造一个 `commit_decision(A)` 再让用户“批准”，也不应该建立一个没有 selection 的假 mutation。

因此 v0 增加一个极小对象：

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

  created_at:
```

它是 durable、immutable 的“需要人类输入”请求，不是 State mutation。

PendingHumanGate 因此允许两种 origin：

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
  status: pending | approved | rejected | answered | cancelled
  created_at:
  resolved_at:
```

两类 Gate 的后续不同：

```text
Authorization Gate approved
→ Policy re-evaluates original immutable State Proposal
```

```text
Intervention Gate answered
→ 根据用户答案生成新的 State Proposal / Working State update
→ 再走 Policy + Reducer
```

这样可以保持三个原则同时成立：

1. Human Gate durable；
2. Reducer 不拥有 Human Gate；
3. Proposal 仍然只描述真正的 desired state transition。

---

# 10. Continuation Decision Priority v0

Policy 不采用“谁分数高谁赢”的黑箱加权。v0 先用确定性优先级。

推荐顺序：

```text
0. Explicit User Control
1. Integrity / Reconciliation
2. Existing Human Gate
3. Provider In-flight
4. Completion Already Applied
5. Maintenance Safety
6. Page Health
7. Context Health
8. Semantic Progress / Completion Candidate
```

展开如下。

## 10.1 Explicit User Control

用户 Pause / typing / manual takeover：

```text
→ PAUSED_USER
```

用户操作永远压过“系统想自动 Continue”。

## 10.2 Integrity / Reconciliation

如果存在：

```text
conversation mismatch
unknown dispatch outcome
state/reference inconsistency
message fingerprint mismatch
```

```text
→ ENTER_RECONCILIATION
```

不能猜。

## 10.3 Existing Human Gate

存在 pending durable gate：

```text
→ PAUSE_FOR_HUMAN
```

## 10.4 Provider In-flight

Provider 正在生成：

```text
→ WAIT
```

绝不同时再发第二条自主消息。

## 10.5 Completion Already Applied

Thread / Run 已经被正式标记 completed：

```text
→ COMPLETED
```

不要因为 Auto Run 开着就继续讨论一个已经完成的任务。

## 10.6 Maintenance Safety

如果当前不处于 Safe Window，Refresh/Rollover 即使“应该发生”，也先 WAIT。

## 10.7 Page Health

页面性能已经退化，但 context 仍健康：

```text
→ REFRESH_SURFACE
```

## 10.8 Context Health

Context pressure / semantic boundary 明显：

```text
→ COMPACT_CONTEXT
→ 必要时 START_ROLLOVER
```

## 10.9 Semantic Progress / Completion Candidate

都没有更高优先级 blocker 时：

```text
advanced
→ CONTINUE_FOCUSED
```

```text
low_progress
→ 改变 next_intent，要求关闭具体问题 / 暴露 blocker
```

```text
candidate_complete
→ REQUEST_COMPLETE
```

```text
blocked by human-only choice
→ create InterventionRequest
→ PendingHumanGate
→ PAUSED_HUMAN
```

原则：

> **信息不足、局部矛盾、需要多推一轮，不自动等于 Human Gate。只有真正越过 authority boundary 或没有任何授权自主路径时才叫人。**

---

# 11. User Preemption：自动化必须随时让路

用户可能在系统准备发送下一轮时开始输入。

因此 user activity 是实时 guard，而不是只在 READY 时检查一次。

## 11.1 未 dispatch 前

如果：

```text
ContinuationDecision 已创建
phase = DISPATCH_PENDING
↓
用户开始输入
```

应：

```text
cancel / suppress pending autonomous dispatch
mode → PAUSED_USER
```

Decision 保留 audit，不删除。

## 11.2 已 dispatch 但未确认时

不能简单再发用户消息或再次 Continue。

```text
→ BLOCKED_RECONCILIATION / Recovery path
```

具体判断依赖 Execution Journal。

## 11.3 Provider 已生成中

用户 Pause 只阻止“下一步自动继续”；当前 response 仍可以正常被观察、ingest 和 settle。

---

# 12. Result Ingestion 顺序必须固定

收到 Assistant Turn 后，不应该立刻依据自然语言最后一句发送下一条。

正确顺序：

```text
Assistant Turn observed
      ↓
Persist Turn + fingerprint
      ↓
INGESTING_RESULT
      ↓
Parse Control Proposal（if any）
      ↓
Create durable State Proposal(s) / InterventionRequest(s)
      ↓
SETTLING_STATE
      ↓
Policy / Reducer settle
      ↓
Read latest Applied State
      ↓
Check PendingHumanGate
      ↓
READY
      ↓
Continuation Decision
```

因此：

> **Continuation 永远基于“最新结果已经进入治理后的状态”，而不是基于尚未落地的自然语言建议。**

---

# 13. Safe Refresh Contract

`REFRESH_SURFACE` 只能在 Safe Refresh Window 执行。

v0 至少要求：

```text
Provider not generating
AND no unresolved dispatch uncertainty
AND latest observed assistant turn is persisted/fingerprinted
AND current Run/Continuation state is durable
AND no unsent user draft
AND current Provider Conversation ref is known
```

Refresh 生命周期：

```text
READY
↓ ContinuationDecision(REFRESH_SURFACE)
MAINTENANCE
↓
Browser reload
↓
Adapter reattach
↓
resolve Provider Conversation
↓
verify last message fingerprint / expected head
├─ match → READY
└─ mismatch / unknown → BLOCKED_RECONCILIATION
```

Refresh 的核心 invariant：

> **Refresh 可以销毁 Browser Session，但不能改变 Logical Thread 的 current Provider Conversation。**

---

# 14. Controlled Rollover Contract

Rollover 不是：

```text
new chat → paste summary → hope for the best
```

也不是一个单独 browser click。

它是一段 **maintenance saga**：

```text
Current Conversation A
      ↓
State settle
      ↓
Stateful Compaction
      ↓
Checkpoint at state vN
      ↓
Context Compiler projection
      ↓
Create Provider Conversation B
      ↓
Inject projection / bootstrap contract
      ↓
Observe & identify B
      ↓
Attach B
      ↓
Switch LogicalThread.current A → B
      ↓
mark A rolled_over
      ↓
READY
```

关键 invariant：

1. A 不删除；
2. A 在 B 被可靠创建/识别以前仍然是 current carrier；
3. current carrier switch 必须可审计；
4. projection 必须来自已 Applied State / immutable evidence refs；
5. rollover failure 不能产生“两个 current conversation”；
6. 如果切换状态不确定，进入 reconciliation。

因此：

> **Rollover 是 Context Boundary Reset；Refresh 是 Browser Surface Reset。**

---

# 15. Page Health 与 Context Health 必须分开

两者不能合成一个“长聊分数”。

## 15.1 Page Health

回答：

> 当前 Browser Surface 是否还能健康交互？

候选信号：

- interaction latency；
- long task；
- DOM/message growth；
- scroll/input delay；
- adapter observer pressure；
- refresh 后性能改善幅度。

具体权重不在 v0 冻结。

## 15.2 Context Health

回答：

> 当前 Provider Conversation 是否仍然是一个好的显式工作边界？

候选信号：

- semantic phase boundary；
- repeated reopening；
- terminology drift；
- recent progress density；
- projection budget pressure；
- manual rollover request；
- rounds/messages since last reset（仅 signal，不是死阈值）。

## 15.3 决策示例

```text
Page degraded + Context healthy
→ Refresh
```

```text
Page healthy + Context pressured
→ Compact / Rollover
```

```text
Page degraded + Context pressured
→ Prefer Controlled Rollover when safe
```

---

# 16. Low Progress Policy：不要一重复就叫用户

`low_progress` 不应该直接：

```text
→ “请问你想怎么办？”
```

v0 推荐至少先尝试一次 focused recovery strategy：

```text
不要继续扩写已有解释；
指出剩余最关键 unknown；
尝试关闭一个具体 Open Question；
如果无法推进，明确 blocker 属于：
- authority choice
- missing evidence
- scope mismatch
- actual completion
```

之后：

- authority choice → Intervention Gate；
- missing evidence 且工具可取得 → 继续自主；
- scope change → gate concrete scope Proposal；
- actual completion → REQUEST_COMPLETE；
- 无法 reconcile → BLOCKED_RECONCILIATION / explicit pause。

具体“连续几轮 low progress”阈值暂不冻结，必须用真实任务 Eval 调。

---

# 17. Completion Contract

Completion 是另一个容易被模型自我宣布的地方。

v0 不接受：

```text
Assistant: “已经完整了。”
→ Thread magically completed
```

Completion 至少分三层：

```text
completion_candidate
→ Completion Policy
→ completion transition applied
→ COMPLETED
```

Completion Policy 可以检查：

- deliverable 是否满足；
- blocking Open Question 是否关闭；
- 是否存在 PendingHumanGate；
- 是否存在 unresolved blocker/reconciliation；
- 是否有明确 completion criteria；
- 是否需要 human confirmation。

如果需要修改 Run operational status：

```text
REQUEST_COMPLETE
→ State Proposal(set_status / completion transition)
→ Policy
→ Reducer
```

只有 ApplyResult 成功后，Continuation mode 才进入 COMPLETED。

---

# 18. Control Proposal 解析失败怎么办

Bootstrap marker 不能成为单点故障。

如果：

```text
assistant response 正常
但 Control Proposal 缺失 / JSON malformed
```

v0 不应该立刻再次发送“继续”。

推荐：

```text
1. 保留完整 assistant Turn
2. 尝试 deterministic parse / repair（仅格式层）
3. 若仍失败，调用独立 evaluator / explicit control-recovery prompt（若配置）
4. 仍无法建立可靠 next-action basis
   → BLOCKED_RECONCILIATION 或 PAUSED_USER
```

> **看不懂控制状态时，不能用自动 Continue 掩盖控制协议已经失效。**

---

# 19. 状态转换主路径

## 19.1 正常自主循环

```text
mode=RUNNING
phase=READY
↓
ContinuationDecision(CONTINUE_FOCUSED)
↓
DISPATCH_PENDING
↓ dispatch proven
AWAITING_PROVIDER
↓ assistant turn observed
INGESTING_RESULT
↓
SETTLING_STATE
↓ no gate / state settled
READY
↓
next decision
```

## 19.2 Human Gate

```text
SETTLING_STATE / READY
↓ PendingHumanGate exists
mode=PAUSED_HUMAN
↓ user resolves gate
Policy / State settle
↓ no other gate
mode=RUNNING (if auto-resume policy permits)
phase=READY
```

默认是否在 human answer 后自动 resume，应由 user/run policy 控制；不要隐藏式决定。

## 19.3 User takeover

```text
any phase
↓ user typing / explicit pause
mode=PAUSED_USER
```

已有 in-flight result 可以继续 ingest，但不自动 dispatch next action。

## 19.4 Refresh

```text
RUNNING + READY
↓ page degraded + safe window
ContinuationDecision(REFRESH_SURFACE)
↓
phase=MAINTENANCE
↓ reattach + head verified
phase=READY
```

## 19.5 Rollover

```text
RUNNING + READY
↓ context pressure + safe window
ContinuationDecision(START_ROLLOVER)
↓
phase=MAINTENANCE
↓ compaction/checkpoint/projection/new conversation
↓ current carrier switch verified
phase=READY
```

## 19.6 Reconciliation

```text
any phase
↓ ambiguous execution / mismatch
mode=BLOCKED_RECONCILIATION
```

Recovery Contract 决定如何回到 RUNNING/READY。

---

# 20. Run Policy / Autonomy Policy

同一个 State Machine 应支持不同自动化强度，而不是把“自动”写死。

v0 可以有：

```yaml
autonomy_policy:
  mode:
    manual | supervised | autonomous

  auto_resume_after_human_gate: false

  allowed_actions:
    - CONTINUE_FOCUSED
    - COMPACT_CONTEXT
    - REFRESH_SURFACE
    - START_ROLLOVER

  unattended_budget:
    max_steps:
    max_wall_time:
```

这些字段的默认数值暂不冻结。

语义：

- `manual`：只观察/维护，不自动发下一轮工作 prompt；
- `supervised`：可以给出 next action，用户确认后 dispatch；
- `autonomous`：在 policy / authority 边界内自动推进。

用户任何时候都可以 Pause / Resume。

---

# 21. Continuation 与 State Delta 的明确接口

Continuation **不直接改 Run State**。

它读取：

```text
latest Applied Run State
AuthorizationResult(s)
PendingHumanGate(s)
ApplyResult(s)
```

需要产生新的状态变化时：

```text
Continuation / Worker
→ durable immutable State Proposal
→ Authority Policy
→ Reducer
```

特别是：

```text
REQUEST_COMPLETE
≠ direct set completed
```

```text
blocked by scope change
≠ direct edit scope
```

```text
human choice answered
→ new Proposal based on answer
```

这样 Continuation 永远不绕过 State governance。

---

# 22. Continuation 与 Execution Journal 的边界

本文只定义：

> “应该做什么”以及“控制器目前处于什么 mode/phase”。

Execution Journal 后续定义：

> “这个 Action Intent 实际发送了吗、Provider 看到了吗、response 是哪一个、refresh/rollover 走到了哪一步、崩溃后如何 reconcile。”

因此：

```text
ContinuationDecision
        ↓
Execution Journal / Dispatcher
        ↓
Observed Runtime Event
        ↓
Continuation State transition
```

`DISPATCH_PENDING` 与 `AWAITING_PROVIDER` 的分界，最终必须由 Journal 的事实证据驱动，而不是 UI 猜测。

---

# 23. v0 必须测试的场景

### A. 正常 Auto Continue

```text
READY
→ CONTINUE_FOCUSED
→ dispatch
→ response
→ state settle
→ READY
```

不能重复发送。

### B. Provider 仍 generating

```text
READY evaluation sees generating
→ WAIT
```

### C. 用户在 dispatch 前开始输入

```text
DISPATCH_PENDING
→ user activity
→ PAUSED_USER
→ suppress autonomous send
```

### D. 用户在 Provider generating 时 Pause

当前 response 可完成并 ingest，但下一轮不自动发送。

### E. Concrete Proposal 需要 Human

```text
AuthorizationResult.requires_human
→ PendingHumanGate
→ PAUSED_HUMAN
```

### F. A/B 产品偏好，尚无唯一 mutation

```text
InterventionRequest
→ PendingHumanGate
→ user answers
→ new State Proposal
```

### G. 页面退化但 Context 健康

```text
→ REFRESH_SURFACE
→ reattach
→ head match
→ READY
```

### H. Refresh 后 head mismatch

```text
→ BLOCKED_RECONCILIATION
```

### I. Context pressure

```text
→ COMPACT_CONTEXT / START_ROLLOVER
```

### J. Rollover 创建 B 失败

A 保持 current；不能出现两个 current carrier。

### K. Rollover B 创建成功但 switch 状态不确定

```text
→ BLOCKED_RECONCILIATION
```

### L. Low progress

先 focused recovery，不立即打扰用户。

### M. Candidate complete

```text
REQUEST_COMPLETE
→ State Proposal / Policy / Reducer
→ applied
→ COMPLETED
```

### N. Control Proposal malformed

不自动裸 Continue；进入 control recovery / reconciliation。

### O. Crash at DISPATCH_PENDING

Continuation 不自行重发。等待 Execution Journal 判断 external side effect 是否已发生。

---

# 24. 当前冻结结论

1. **Continuation Controller 以 Logical Thread 为控制单元。**
2. **Control Mode 与 Execution Phase 正交，避免状态爆炸。**
3. **ContinuationDecision 必须 durable；Decision ≠ Execution Fact。**
4. **Provider in-flight 时绝不再自主发送第二条消息。**
5. **用户操作拥有最高 preemption priority。**
6. **Continuation 基于 settled/applied state 决策，不直接读一句自然语言就继续。**
7. **Human Gate 有两种 origin：State authorization approval 与 pre-state InterventionRequest。**
8. **Continuation 不直接修改 Run State；所有 state mutation 继续走 Proposal → Policy → Reducer。**
9. **Refresh 只重建 Browser Surface；Rollover 重建显式 Context Boundary。**
10. **Safe Refresh / Rollover 必须等待 safe window。**
11. **Rollover 是 maintenance saga，不是一个 browser click。**
12. **Page Health 与 Context Health 分开。**
13. **Low Progress 先自主 focused recovery，不直接 Human Gate。**
14. **Completion 必须经过 Completion Policy / State transition，不接受模型自我宣布。**
15. **执行历史不确定时进入 reconciliation，不通过重复发送猜测。**
16. **Execution Journal 负责“实际发生了什么”，Continuation 负责“下一步应该做什么”。**

---

# 25. 下一步接口

下一篇应当进入：

> **Execution Journal & Recovery Contract v0**

重点不是再扩 Continuation action，而是钉死：

```text
ContinuationDecision
→ planned execution
→ provider/browser side effect
→ observation
→ reconciliation
→ idempotent recovery
```

尤其要解决：

- send 到底有没有发生；
- refresh 前后如何对齐；
- rollover saga 每一步如何恢复；
- browser crash 后怎样避免 duplicate Continue；
- 用户与自动化同时操作时如何确定最终事实。

在写 Recovery Contract 前，本文继续保持 **Design Candidate v0**；应先与 Object Model / State Delta 做一次接口对审。
