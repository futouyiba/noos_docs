# NOOS Harness｜Multi-Conversation Orchestration Role Model v0

> **状态**：Design Candidate v0  
> **日期**：2026-08-27  
> **依赖基线**：[NOOS Harness Overview v0.2](overview.md) · [Runtime Object Model & Authority Model v0](runtime-object-authority-model.md) · [State Delta + Reducer Contract v0](state-delta-reducer-contract.md)  
> **相关 Candidate**：[Continuation State Machine v0](continuation-state-machine.md)  
> **目标**：定义一个 Run 内的多对话工作如何按“总控 / 设计 / 审核 / 集成”四种 Logical Thread Role 进行分工、流转和收敛。

---

## 0. 一句话说明

NOOS 不应该把多对话编排理解为“同时打开几个 ChatGPT 窗口”。

真正需要的是：

> **一个 Run 拥有多个语义明确的 Logical Thread；不同 Thread 分别负责规划、设计、审核和集成，而 Provider Conversation 只是这些 Thread 的可替换执行载体。**

四种核心 Role：

```text
Control      总控
Design       设计
Review       审核
Integration  集成
```

它们回答四个不同的问题：

| Role | 核心问题 |
|---|---|
| Control | 现在应该做什么？ |
| Design | 这个局部问题应该怎么解决？ |
| Review | 当前 Candidate 哪里有问题？ |
| Integration | 面对设计与审核意见，应该形成什么新的统一 Candidate？ |

---

# 1. 第一条 invariant：Control Thread ≠ Continuation Controller

这两个名字很容易混，但属于完全不同层。

## Control Thread

是一个 **AI 工作角色 / Logical Thread Role**。

它可以：

- 理解 Run 当前状态；
- 规划下一阶段；
- 创建/结束 Design、Review、Integration 工作；
- 汇报当前进度；
- 识别何时需要用户选择；
- 提议 Run-level 下一步。

它本质上仍然是模型参与的 reasoning thread。

## Continuation Controller

是 **NOOS Harness Runtime 的控制模块**。

它负责：

- 当前这个 Logical Thread 是否允许自动继续；
- 是否处于 READY / AWAITING_PROVIDER / MAINTENANCE；
- 是否该 Refresh / Rollover / Pause；
- 是否存在 dispatch guard / runtime blocker。

它不是一个 Chatbot 人格，也不负责“想方案”。

因此：

```text
Control Thread
= orchestration reasoning role

Continuation Controller
= runtime execution-control mechanism
```

> **所有四种 Logical Thread，包括 Control Thread 自己，都受 Continuation Controller 管理。**

---

# 2. 四 Role 是 Thread Role，不是“四个固定聊天窗口”

必须避免：

```text
总控 Chat
设计 Chat
审核 Chat
集成 Chat
```

被误解成永远只有四个 ChatGPT URL。

真实模型是：

```text
Run
│
├─ Control Thread
│  └─ Provider Conversation A → B → ...
│
├─ Design Thread #1
│  └─ Provider Conversation C → D → ...
├─ Design Thread #2
│  └─ Provider Conversation E
│
├─ Review Thread #1 / Domain
│  └─ Provider Conversation F
├─ Review Thread #2 / Product
│  └─ Provider Conversation G
├─ Review Thread #3 / Architecture
│  └─ Provider Conversation H
│
├─ Integration Thread #1
│  └─ Provider Conversation I
└─ Integration Thread #2
   └─ Provider Conversation J
```

所以：

> **4 Role Types ≠ 4 Provider Conversations。**

NOOS UI 应主要向用户展示 Role / Thread / Work Item，而不是要求用户管理大量实际 Chat URL。

---

# 3. 总体闭环：Design → Review → Integration → State → Control

一个典型工作循环：

```text
                 ┌──────────────┐
                 │ Control      │
                 │ 识别下一问题  │
                 └──────┬───────┘
                        ↓
                 ┌──────────────┐
                 │ Design       │
                 │ 形成 Candidate│
                 └──────┬───────┘
                        ↓
                Design Snapshot
                        ↓
          ┌─────────────┼─────────────┐
          ↓             ↓             ↓
     Review/Domain  Review/Product  Review/Arch...
          └─────────────┼─────────────┘
                        ↓
                  Review Issues
                        ↓
                 ┌──────────────┐
                 │ Integration  │
                 │ 逐项裁决/整合 │
                 └──────┬───────┘
                        ↓
             Integrated Candidate
             + State Proposal(s)
                        ↓
                  Policy / Reducer
                        ↓
                 Applied Run State
                        ↓
                 ┌──────────────┐
                 │ Control      │
                 │ 重新评估下一步│
                 └──────────────┘
```

关键原则：

> **Thread 之间传递结构化工作产物，而不是互相复制整段聊天历史。**

---

# 4. Control Thread｜总控

## 4.1 职责

Control Thread 回答：

> **这个 Run 现在应该做什么？**

它关注：

- Goal / Deliverable；
- Scope；
- 当前 Phase；
- Committed State；
- Open Questions；
- 活跃 Work Item；
- 各 Logical Thread 状态；
- Review / Integration 是否完成；
- Pending Human Gate；
- Completion readiness。

## 4.2 不负责什么

Control Thread 默认不应该：

- 长时间深入推导某个领域设计；
- 自己承担所有 Review dimension；
- 把 Reviewer disagreement 直接当事实裁决；
- 绕过 Policy / Reducer 修改 Committed State；
- 直接替代 Integration 对复杂冲突做全文重写。

> **Control 负责路由和阶段判断，不负责成为“最聪明的万能 Agent”。**

## 4.3 典型输入

```yaml
control_projection:
  run_goal:
  deliverable:
  scope:
  current_phase:

  committed_state_summary:
  open_questions: []
  active_work_items: []
  thread_statuses: []
  pending_human_gates: []
  latest_integration_result:
```

## 4.4 典型输出

Control Thread 不直接 dispatch browser action，而是形成 orchestration intent，例如：

```text
CREATE_DESIGN_WORK
REQUEST_REVIEW
START_INTEGRATION
ASK_HUMAN
MARK_PHASE_CANDIDATE_COMPLETE
CONTINUE_EXISTING_THREAD
```

正式的 Run State mutation 仍然走：

```text
Control reasoning
→ State Proposal
→ Policy
→ Reducer
```

## 4.5 生命周期

Control Thread 默认 **Run-scoped**：

```text
Run created
→ Control Thread created
→ persists across many cycles
→ may rollover Provider Conversation
→ Run completed
```

它通常是用户最主要面对的 Thread。

---

# 5. Design Thread｜设计

## 5.1 职责

Design Thread 回答：

> **这个明确的局部问题应该怎么解决？**

适合承担：

- 第一性原理推导；
- 方案生成；
- 方案比较；
- 机制拆解；
- 发现 missing concept；
- 深挖一个 Open Question；
- 收敛为 Design Candidate。

## 5.2 Design Thread 必须有窄 Brief

不能只给：

> “继续设计整个系统。”

最小 Design Brief：

```yaml
design_brief:
  work_item_id:
  objective:
  question:
  deliverable:

  relevant_committed_decisions: []
  relevant_constraints: []
  relevant_rejected: []
  relevant_open_questions: []
  source_refs: []

  in_scope: []
  out_of_scope: []
```

## 5.3 Design Thread 可以深入，但不能越权

它可以跑几十轮，并通过 Conversation Rollover 保持同一个 Logical Thread。

但它不能：

- 宣布整个 Run 完成；
- 改 Run Scope；
- 把自己的 Candidate 直接升级为 Committed Decision；
- 自己决定“不需要 Review”；
- 修改其他 Thread 的任务。

## 5.4 输出

建议输出：

```yaml
design_candidate:
  id:
  work_item_id:
  base_state_version:
  summary:
  proposed_model:
  rationale:
  known_risks: []
  unresolved_questions: []
  evidence_ref_ids: []
  candidate_state_proposal_ids: []
```

Design Candidate 是 Review / Integration 的输入，不天然等于 Committed State。

## 5.5 生命周期

Design Thread 更适合 **topic/work-item scoped**。

一个重要专题可以长期存在并跨多个 Provider Conversation；专题关闭后 Thread completed。

---

# 6. Review Thread｜审核

## 6.1 职责

Review Thread 回答：

> **在一个明确的审核维度下，当前 Candidate 哪里可能不成立？**

Review 的默认立场不是“再设计一套”，而是：

```text
给定 Candidate
→ 找 blocker / contradiction / hidden cost / unsupported claim
```

## 6.2 Review 必须正交

推荐先用：

```text
domain
product
architecture
production
```

也可以按 Run 类型扩展，例如：

```text
security
legal
performance
economy
ux
```

但一个 Review Thread 应尽量只拥有一个主要 dimension。

## 6.3 Review Snapshot 必须 immutable

Reviewer 不能边审一个不断变化的 Design Thread。

它应该吃：

```yaml
review_snapshot:
  id:
  base_state_version:
  design_candidate_id:
  design_candidate_fingerprint:
  review_dimension:
  review_contract:
  source_refs: []
  created_at:
```

所以 Reviewer 输出天然带有 stale semantics。

## 6.4 Review 输出不是自然语言长评，而是一组 Issue

```yaml
review_issue:
  id:
  review_snapshot_id:
  base_state_version:
  dimension:
  severity:

  target:
  claim:
  reasoning:
  evidence_ref_ids: []
  suggested_action:
```

建议 severity 至少：

```text
blocker
major
minor
note
```

## 6.5 Reviewer 不能直接改 Candidate / State

Reviewer 默认只能：

```text
raise issue
suggest change
request evidence
mark no blocker
```

它不能：

```text
review → rewrite committed state
```

否则 Review 和 Integration 又混在一起。

## 6.6 生命周期

Review Thread 默认 **snapshot-scoped**。

```text
Review Snapshot RS-17
→ Reviewer Thread
→ Review Issues
→ Completed
```

下一版 Candidate 更适合创建新的 Review Thread，而不是让同一个 Reviewer 对话无限增长。

这有三个好处：

1. 上下文更干净；
2. stale boundary 明确；
3. 不容易让 Reviewer 被旧版本论证锚定。

---

# 7. Integration Thread｜集成

## 7.1 职责

Integration Thread 回答：

> **面对 Design Candidate 与一组 Review Issues，应形成什么新的统一 Candidate？**

它承担的是 reconciliation / adjudication reasoning，而不是权威本身。

## 7.2 输入

```yaml
integration_packet:
  integration_cycle_id:
  base_state_version:

  design_candidate_id:
  review_snapshot_ids: []
  review_issue_ids: []

  relevant_committed_state:
  relevant_constraints: []
  relevant_rejected: []
  source_refs: []
```

## 7.3 对每条 Issue 必须显式处理

不能只输出一篇“综合后新版方案”。

至少记录：

```text
ACCEPT
REJECT
PARTIAL_ACCEPT
DEFER
NEEDS_HUMAN
OBSOLETE
```

例如：

```yaml
adjudication:
  issue_id: RI-031
  disposition: PARTIAL_ACCEPT
  reason:
  resulting_change:
  evidence_ref_ids: []
```

这样未来能回答：

> Reviewer 提的那个 blocker 后来去哪了？

## 7.4 Integration 输出

```yaml
integration_result:
  id:
  integration_cycle_id:
  base_state_version:

  adjudications: []
  integrated_candidate:
  state_proposal_ids: []
  unresolved_issues: []
  intervention_request_ids: []
```

## 7.5 Integration 不是 Authority

即使 Integration 认为 Reviewer A 是对的：

```text
Integration Result
→ State Proposal
→ Authority / Promotion Policy
→ Reducer
→ Committed State
```

不能：

```text
Integrator says so
→ automatically committed
```

## 7.6 生命周期

Integration 更适合 **cycle-scoped**。

```text
Integration Cycle #7
→ fresh Integration Thread
→ result
→ completed
```

下一轮 Design / Review 后可以创建新的 Integration Thread。

这样它不需要携带所有历史 integration reasoning，只读取当前 Committed State + 本轮输入。

---

# 8. 四 Role 的默认生命周期

| Role | 默认生命周期 | 是否适合长对话 | 是否常用 fresh conversation |
|---|---|---:|---:|
| Control | Run-scoped | 是 | rollover 时换 carrier |
| Design | Topic / WorkItem-scoped | 是 | 必要时 rollover |
| Review | Snapshot-scoped | 通常否 | **是** |
| Integration | Integration-cycle-scoped | 通常否 | **是** |

因此 Harness 不应追求“所有 Thread 都长期保活”。

> **Durable 的是语义工作对象；具体对话是否长寿由 Role 决定。**

---

# 9. 四 Role 的 State 权限必须保持一致

Role 不是 Authority shortcut。

默认：

```text
Control
Design
Review
Integration
```

都不能绕过：

```text
Proposal
→ Policy
→ Reducer
```

角色差异体现在：

- 它被允许提出什么 Proposal；
- 默认 delegated authority 多大；
- 读取什么 Context Projection；
- 输出什么工作产物。

而不是：

```text
Integrator = 超级用户
Control = root
```

---

# 10. Context Projection 必须按 Role 编译

同一个 Run State，不应该原样塞给所有 Thread。

## 10.1 Control Projection

强调：

```text
Goal
Phase
Open Questions
WorkItem status
Thread status
Pending Gates
latest Integration result
```

少给局部推导噪音。

## 10.2 Design Projection

强调：

```text
Design Brief
relevant committed decisions
relevant constraints/rejections
current frontier
relevant evidence
```

不需要看到所有其他 Reviewer 的历史。

## 10.3 Review Projection

强调：

```text
immutable Design Snapshot
review dimension contract
relevant source evidence
known constraints
```

Reviewer 不应该默认读取 Design Thread 的全部 reasoning history，否则容易跟着设计者的论证走。

## 10.4 Integration Projection

强调：

```text
Current Committed State
Design Candidate
Review Issues
Issue evidence
integration rules
```

也不需要把所有 Reviewer conversation transcript 带进去。

这正是 Context Compiler 的 purpose-built projection。

---

# 11. 用户交互：默认面向 Control，但允许直接进入任意 Thread

最终产品体验推荐：

> **用户拥有一个 Run，而不是拥有二十个 Chat。**

默认主入口是 Control Thread。

用户可以说：

```text
继续。
这一轮先别做 Review。
Product Reviewer 再看一次。
方案 B。
这个专题另外开一个 Design Thread。
```

Control 负责把这些需求路由成 Run-level orchestration change。

但用户仍应能直接打开某个 Design / Review / Integration Thread 阅读或接管。

## 11.1 用户直接修改某个 Design Thread

该 Thread ingest 用户 Turn，并继续按 State Delta / Continuation contract 工作。

## 11.2 用户在 Control 中做产品选择

可通过 InterventionRequest / HumanGateResolution 形成 durable input，再转成相关 State Proposal / WorkItem change。

---

# 12. 不建议现在增加第五种主要 Role

真实工作还经常需要：

```text
Research
Evidence collection
Source verification
```

但 v0 不急着增加 `Research Thread` 作为第五核心角色。

先建模成：

```text
Design / Review / Integration
→ Evidence Request
→ Tool / Research Job
→ SourceRef / EvidenceRef
→ 返回调用 Thread
```

只有当真实使用证明“Research 自身需要长期 reasoning continuity”时，再提升为新的 Logical Thread Role。

这样可以避免 Role taxonomy 过早膨胀。

---

# 13. 一个建议新增的编排对象：WorkItem

四 Role 一旦出现，仅靠 `LogicalThread.role` 还不够回答：

> “这些 Thread 现在共同在处理哪个问题？”

因此建议候选增加一个很薄的 `WorkItem`，但暂不修改 Runtime Baseline。

```yaml
work_item:
  id:
  run_id:
  title:
  objective:
  status:

  owner_control_thread_id:
  design_thread_ids: []
  review_thread_ids: []
  integration_thread_ids: []

  base_state_version:
  created_at:
  completed_at:
```

它不是 Task Management 大系统，只是四 Role 的 correlation object。

典型：

```text
WorkItem W-17
“Feeding Motivation 应该是 State 还是 Derived Variable？”

├─ Design D-3
├─ Review R-8 / Domain
├─ Review R-9 / Architecture
└─ Integration I-4
```

是否正式纳入 Object Model，应由后续接口 Review 决定。

---

# 14. Orchestration 不应依赖大家互相聊天

禁止默认拓扑：

```text
Design Chat ↔ Reviewer Chat ↔ Integration Chat ↔ Control Chat
```

这种 peer-to-peer 自由互聊会造成：

- ownership 不清；
- 信息复制；
- stale 状态难判断；
- 冲突难审计；
- context 迅速膨胀。

推荐：

```text
Thread
↓ structured output / immutable snapshot
NOOS-owned object
↓ purpose-built projection
Next Thread
```

即：

> **Agent-to-Agent communication 默认通过 Harness-owned artifacts/state，而不是直接 conversation-to-conversation transcript relay。**

---

# 15. Role-specific completion

四种 Thread 的“完成”含义不同。

## Control Complete

通常只有 Run 真正结束时完成。

## Design Complete

该 Design Brief 已形成足以进入 Review / Integration 的 Candidate，或明确 blocker。

## Review Complete

对指定 Review Snapshot 和 dimension 已给出完整 Issue Set / No-Blocker 结论。

## Integration Complete

本轮 Review Issues 已全部获得 disposition，并形成 Integrated Candidate / Proposal / Intervention。

因此不能使用一个统一：

```text
assistant says done
→ thread complete
```

Completion Policy 应读取 `logical_thread.role`。

---

# 16. Review 与 Integration 不应该反复互相迭代到无穷

一个自然风险是：

```text
Design
→ Review
→ Integration
→ Review again
→ Integration again
→ ...
```

v0 建议由 Control Thread 明确 cycle boundary。

例如：

```text
Cycle 7
Design Candidate DC-17
→ Review Set RS-17
→ Integration IR-17
→ State Applied
→ Control re-evaluates
```

如果 Integration 后仍有 blocker：

- 需要重新设计 → 新 Design Cycle；
- 只是验证修改 → 新 Review Snapshot；
- 需要用户选择 → Human Gate；
- 已满足 → WorkItem close。

不要让 Review 与 Integration 自己无限 ping-pong。

---

# 17. 最小 Role Schema Candidate

建议 `LogicalThread` 后续可扩展：

```yaml
logical_thread:
  id:
  run_id:

  role:
    control | design | review | integration

  objective:
  status:
  current_provider_conversation_id:

  work_item_id:

  role_config:
    # design
    design_topic:

    # review
    review_dimension:
    review_snapshot_id:

    # integration
    integration_cycle_id:

  created_at:
  completed_at:
```

是否真的做成一个 union schema，留到 Object Model 下一版本；这里先冻结语义，不急着实现字段。

---

# 18. 四角色默认权限矩阵

| 能力 | Control | Design | Review | Integration |
|---|---:|---:|---:|---:|
| 深入生成局部方案 | 不主责 | **主责** | 否 | 可做必要修补 |
| 创建 Design Candidate | 可请求 | **是** | 否 | 可生成 Integrated Candidate |
| Raise Review Issue | 可请求 | 可自检但非正式 | **是** | 可标记新 blocker |
| Adjudicate Review Issue | 否 | 否 | 否 | **主责** |
| 规划下一 WorkItem | **主责** | 建议 | 建议 | 建议 |
| 直接改 Committed State | 否 | 否 | 否 | 否 |
| 提交 State Proposal | 是 | 是 | 通常只 Issue | **是** |
| 触发 Human Gate request | 是 | 是 | 是 | **是** |
| 宣布 Run Complete | 只能 request | 否 | 否 | 否 |

所有 State Proposal 仍由 Policy / Reducer governance。

---

# 19. v0 典型场景

## A. 单一设计专题

```text
Control identifies Q-014
→ Design Thread
→ Candidate
→ Control accepts no review required by policy
→ State Proposal
```

不是每个 Design 都必须四角色全跑一遍。

## B. 高风险设计专题

```text
Control
→ Design
→ Domain + Product + Architecture Review
→ Integration
→ State Proposal
→ Control
```

## C. Reviewer 提出 blocker

```text
Review Issue blocker
→ Integration ACCEPT
→ needs redesign
→ Control opens new Design Cycle
```

Reviewer 自己不负责修到底。

## D. Reviewer 冲突

```text
Reviewer A says ACCEPT MODEL X
Reviewer B says MODEL X breaks product strategy
→ Integration attempts adjudication
→ if authority/preference boundary remains
→ InterventionRequest / Human Gate
```

## E. Integration 后 State 已变

旧 Review Snapshot 不继续作为当前事实；需要复审时产生新 Snapshot。

---

# 20. 当前冻结结论

1. **v0 采用四种核心 Logical Thread Role：Control / Design / Review / Integration。**
2. **Control Thread 是 AI orchestration role；Continuation Controller 是 Runtime mechanism，二者严格分离。**
3. **四 Role Types 不等于四个 Provider Conversations。**
4. **Control 负责阶段规划和路由，不承担所有深度 reasoning。**
5. **Design 负责把窄问题做深并形成 Candidate。**
6. **Review 负责在一个正交 dimension 下寻找问题，不直接重写设计或 State。**
7. **Integration 负责逐项 adjudicate Review Issues 并形成 Integrated Candidate，但不是 Authority。**
8. **所有角色修改 State 都必须走 Proposal → Policy → Reducer。**
9. **Review 默认 snapshot-scoped；Integration 默认 cycle-scoped；Design 默认 topic-scoped；Control 默认 Run-scoped。**
10. **Context Compiler 必须按 Role 生成不同 Projection。**
11. **Agent-to-Agent communication 默认通过 Harness-owned structured artifacts/state，不直接转发完整聊天。**
12. **用户默认主要面对 Control Thread，但可以直接查看/接管任一 Thread。**
13. **Research/Evidence v0 先作为 Job/Tool 能力，不急着升第五 Role。**
14. **WorkItem 是值得进一步验证的薄 correlation object，但当前保持 Candidate，不单方面修改 Runtime Baseline。**
15. **Review ↔ Integration 不允许自行无限 ping-pong；cycle boundary 由 Control 管理。**

---

# 21. 下一步接口审查

本文暂时保持 **Design Candidate v0**。

在升 Baseline 前，应重点检查：

1. `logical_thread.role` 是否需要正式进入 Runtime Object Model；
2. `WorkItem` 是否真有必要成为一等 durable object，还是可以先由 Run State 表达；
3. Control Thread 的 orchestration intent 与 ContinuationDecision 是否会发生职责重叠；
4. Review Snapshot / Design Candidate / Integration Result 的 immutable/version contract；
5. Integration 对 Review Issue 的 adjudication 是否需要独立 durable object；
6. 多 Thread 并发推进时，对 `base_state_version` 的 stale/rebase 策略；
7. 用户在 Control Thread 下达指令时，如何安全影响另一个正在运行的 Design Thread。

这些问题通过后，四角色模型才能真正进入 Multi-Conversation Orchestration Baseline。
