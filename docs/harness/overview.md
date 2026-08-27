# NOOS Harness：把 Chatbot 从长对话变成可持续运行的 AI 工作执行器

> **状态**：Design Baseline v0.2  
> **日期**：2026-08-27  
> **项目**：NOOS  
> **中文工作名**：怒思（候选）  
> **主题**：Run Continuity / Context Control / State Authority / Browser Chatbot Harness

---

## 0. 一句话说明

今天用 ChatGPT、Claude 这类 Chatbot 做复杂设计，真正限制工作的往往已经不是“模型不会回答”，而是：**工作无法被持续、稳定、可控地运行。**

一个复杂专题可能需要连续推进几十轮。用户需要不断手工发送“继续”；对话越来越长以后网页可能变卡；模型也可能开始重复、漂移、忘记已关闭的问题；如果再开几个独立对话做领域审查、产品审查、实现审查，用户还要人工搬运上下文和意见。

NOOS Harness 的目标不是再造一个 ChatGPT，而是把这些原本靠用户手工维持的过程，变成一个由 NOOS 管理的长期 Run：

> **NOOS 持有工作的连续性、任务状态和显式工作上下文；Chatbot 负责阶段性推理。Provider Conversation 可以被替换，Browser Session 可以被刷新和销毁，而 Run 始终连续。**

---

# 1. 我们真正要解决的，不是“自动点继续”

典型复杂设计流程更像这样：

```text
提出问题
  ↓
推第一层
  ↓
继续
  ↓
发现遗漏
  ↓
继续
  ↓
审查 ownership / double-count
  ↓
继续
  ↓
检查玩家策略空间
  ↓
……
```

二三十轮并不罕见。

其中大量“继续”并不包含新的用户判断。用户真正需要介入的时刻，通常只是少数几个：

- 两个方案都合理，需要产品选择；
- 需要改变任务范围；
- 要推翻已经确认的重要决定；
- 要执行外部写入或不可逆动作。

因此，“自动 Continue”只是表层需求。真正的产品问题是：

> **怎样让一项 AI 工作在用户不持续盯着页面的情况下，仍然能向前推进、保持状态、处理故障，并在真正需要人类权威时暂停。**

---

# 2. 需求可以分成四层

```text
┌──────────────────────────────────────┐
│ 4. Workflow Orchestration            │
│ Multi-review / Fan-out / Human Gate  │
├──────────────────────────────────────┤
│ 3. Context Harness                   │
│ State / Compaction / Projection      │
├──────────────────────────────────────┤
│ 2. Session Continuity Runtime        │
│ Continue / Refresh / Rollover        │
├──────────────────────────────────────┤
│ 1. Provider & Browser Adapter        │
│ ChatGPT / Claude / browser surface   │
└──────────────────────────────────────┘
```

旁边还有独立的 Context Source：

```text
Vault / Wiki / Notion / Git / Files / Reference
```

它们不是 Runtime 本身。

---

# 3. 最重要的对象边界：Run ≠ Provider Conversation ≠ Browser Session

旧式 Chatbot 的默认心智模型是：

```text
Conversation = 这项工作
```

NOOS Harness 改成：

```text
Run = 用户真正拥有的长期工作
Logical Thread = 这项工作中的一个持续角色 / 思考维度
Provider Conversation = 某个平台保存的一段对话载体
Browser Session = 某次网页运行与挂接环境
```

例如：

```text
Run：World Condition → Fish Response
│
├─ Logical Thread：Main Design
│   ├─ Provider Conversation A (ChatGPT)
│   ├─ Provider Conversation B (ChatGPT)
│   └─ Provider Conversation C (Claude)
│
├─ Logical Thread：Domain Review
│   └─ Provider Conversation D
│
└─ Logical Thread：Production Review
    └─ Provider Conversation E

Browser Session / Tab / Adapter Attachment
    ↳ 只是某一时刻承载上述 Conversation 的临时执行表面
```

因此：

- **Run 是 durable，NOOS-owned。**
- **Logical Thread 是 durable，NOOS-owned。**
- **Provider Conversation 是 replaceable carrier，通常由外部平台持久化，并且必须保留 provenance。**
- **Browser Session / Adapter Attachment 才是真正 disposable 的 runtime resource。**

v0 再增加一个简单 invariant：

> **一个 Logical Thread 同一时刻至多有一个 current Provider Conversation。**

Reviewer fan-out 应通过多个 Logical Thread 实现。

详细定义见：[`runtime-object-authority-model.md`](runtime-object-authority-model.md)。

---

# 4. NOOS 拥有的不是“全部聊天记录”，而是 Working State

只把完整 conversation 存下来，并不等于拥有上下文。

Harness 真正需要维护的是：

```text
Raw Transcript
      ↓
State Extraction / State Delta
      ↓
Run Working State
      ↓
Context Compiler
      ↓
Next Execution Projection
```

Run 内信息分三层：

### Durable Context

Goal / Deliverable / Scope / Constraints / Committed Decisions / Rejected Decisions / 重要 provenance。

### Working Context

Hypotheses / Open Questions / Current Frontier / Active Branch / Reviewer Issues / 当前推导状态。

### Ephemeral Context

局部例子、临时反驳、exploratory branch、最近几轮局部 reasoning trace。

原则不是“什么都不能忘”，而是：

> **默认 Raw Conversation 只是可回溯证据；只有被明确提升到 State 的内容才获得持续工作语义。**

---

# 5. Operational Authority 与 Epistemic Authority 必须分开

Run State 可以决定“工作怎么继续”，但不能因为模型总结过一次，就变成所有外部事实的最终真理。

## Operational Authority

由 Run State 管理，例如：Goal、Scope、Committed Decision、Open Question、Frontier、Next Action。

## Epistemic Authority

外部事实由来源证据决定，例如：Notion Current Production Fact、GitHub 当前代码、正式规范、用户明确提供的 normative constraint。

链路应是：

```text
Source observation
      ↓
Evidence relationship
      ↓
Run State interpretation
```

而不是：

```text
Source
↓
模型总结一次
↓
State 永远变成真理
```

这里进一步区分两个对象：

```text
SourceRef
= 我观察了什么、哪个版本、何时看到

EvidenceRef
= 这个 Source observation 对哪个 Claim 提供什么证据，具有什么 authority role
```

因此 `claim_kind` / `authority_role` 属于 EvidenceRef，不属于 SourceRef 本体。

同时：

> **Source observation 的内容身份必须 immutable。重新观察同一逻辑来源时创建新 SourceRef，而不是覆盖旧 observation。**

这样旧 Decision 的 provenance 不会被今天的新版本篡改。

一句话：

> **Run State 是 working authority，不是所有事实的 ultimate source of truth。**

---

# 6. Compaction 不是 Summary，而是 Stateful Compaction

普通“总结以上对话”很容易把 Confirmed、Hypothesis、Rejected、Open Question、Evidence 压成一篇流畅但失真的文字。

真正需要的是：

> **Stateful Compaction：从最近一段轨迹中提取结构化 Proposal / State Delta，并更新短期 Carry Context。**

Raw Transcript 仍然归档，但退出 active working context。

---

# 7. LLM 不能直接重写 State：Proposal → Policy → Reducer

仅仅有 Reducer 还不够。

Reducer 能保证 schema 和状态一致性，却不能回答：

> “这个模型有没有资格把一个 hypothesis 直接升级为 Committed Decision？”

因此正确链路是：

```text
LLM / Tool
   ↓
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
          ↓
       ApplyResult / Audit
```

原则：

> **LLM proposes; Policy authorizes; Reducer applies; NOOS records.**

`Proposal` 本身就是请求；Operation 描述期望发生的最终 State Transition。因此不再需要 `propose_decision` 这类二次“提议操作”。

v0 同时冻结：

> **Proposal atomicity → Authorization atomicity → Reducer atomicity。**

`requires_human` 由 Policy 创建 durable AuthorizationResult / PendingHumanGate；Reducer 不负责 Human Gate。

详细 Contract 见：[`state-delta-reducer-contract.md`](state-delta-reducer-contract.md)。

---

# 8. Context Store ≠ Context Projection

NOOS 知道的东西，不等于每次执行都应该让模型看到。

```text
Vault / Sources
      +
Run State
      +
Current Action
      ↓
Context Compiler
      ↓
Purpose-built Context Projection
```

Context Compiler 的价值在选择、裁剪、排序、维护 provenance 和控制 working-set 大小。

Pending Proposal / Pending Human Gate 可以显式展示，但不能伪装成 Committed State。

---

# 9. Context Control 的真实边界

NOOS 无法完全控制 ChatGPT / Claude 的 context window，也通常不能可靠读取 system prompt、account memory、provider-side summary、tool state 和 provider policy。

因此不使用“Hard Context Control”。

## Same Conversation：Soft Guidance

NOOS 可以重申 constraint、注入 relevant decision、检测 drift、给出 focused next action，但无法真正删除既有 history。

## New Provider Conversation：Controlled Context Reset

Rollover 后，NOOS 可以更强地决定显式投喂的工作历史：

```text
Harness Contract
+ Goal
+ Applied State
+ Relevant Evidence
+ Carry Context
+ Next Action
```

> **Rollover 的价值不只是性能，而是获得更清晰的 explicit-history boundary。**

---

# 10. Session Continuity：Refresh 与 Rollover 是两类不同治疗手段

## Safe Refresh

重建浏览器运行环境。Provider Conversation 不变。

## Conversation Rollover

重建模型工作的显式上下文边界。Provider Conversation 更换，但仍属于同一个 Logical Thread / Run。

```text
Performance degradation
→ Prefer Refresh

Context / semantic pressure
→ Compact + Rollover
```

Round Count 只能是 signal，不能变成固定阈值。

页面卡顿的具体机理目前只视为 Candidate Mechanism：

> **instrument first, optimize second.**

---

# 11. Continuation 不是“继续”，而是 Next Action Policy

v0 动作可以保持很小：

```text
CONTINUE_FOCUSED
COMPACT
REFRESH
ROLLOVER
ASK_HUMAN
COMPLETE
```

`CONTINUE_FOCUSED` 应表达当前真正 frontier，而不是裸发“继续”。

推荐决策优先级：

```text
1. User Activity
2. State Integrity
3. Authority Boundary
4. Completion
5. Page Health
6. Context Health
7. Progress
```

---

# 12. Recovery 不能只靠 Checkpoint：还需要 Execution Journal

只有 `state_version` / `checkpoint_id` 不足以可靠恢复。

系统必须区分：

```text
Provider side effect 是否发生
Assistant result 是否观察到
reconciliation 是否完成
State Proposal / Delta 是否接受与应用
```

因此：

```text
State Store = 现在是什么
Execution Journal = 执行过程中发生了什么
```

具体 event/status 结构留给 `Execution Journal & Recovery Contract v0`，不在 Overview 中提前冻结成单轴状态机。

---

# 13. Harness Control Block 只是 Bootstrap Transport

MVP 可以让 Worker 在输出末尾追加机器可读 Control Proposal，但架构 contract 是 **Control Proposal**，HTML marker 只是一种 transport implementation。

未来可替换为 provider structured output、独立 controller call、local evaluator 或 Shadow Controller。

---

# 14. Multi-Conversation Review 建立在 Harness 之上

```text
Run State v37
      ↓
Review Snapshot RS-008
      ↓
Orthogonal Review Threads
      ↓
Structured Review Issues
      ↓
Owner / Policy Adjudication
      ↓
Proposal / State Delta
```

Review Issue 至少记录 `base_state_version` / `review_snapshot_id`，用于识别 stale review。

---

# 15. Harness 与 NOOS Hub 的关系

```text
NOOS Hub
├─ Vault / Artifact Store
├─ Context Broker
├─ Harness Runtime
└─ Tool Router

Browser Shuttle
└─ Provider / Chatbot Adapter
```

> **Harness Runtime 是 Hub 的 Execution subsystem；Shuttle 是浏览器侧执行与连接层。**

v0 不制造第二个本地 daemon。

---

# 16. MVP 拆成五个可验证里程碑

### M0 — Run Continuity Proof

验证工作能否从网页生命周期中解耦。

### M1 — Controlled Rollover

验证 Stateful Compaction + Context Projection + Conversation Rollover 本身有没有价值。

### M2 — Autonomous Continuation

加入 CONTINUE_FOCUSED / ASK_HUMAN / COMPLETE。

### M3 — Performance Self-Healing

加入 performance telemetry / safe refresh / auto reattach。

### M4 — Reviewer Orchestration

加入 review snapshot / orthogonal reviewer / issue adjudication。

---

# 17. Eval：必须拆开机制贡献

实验至少比较：

```text
A. Long Chat
B. Human-managed Rollover
C. NOOS Controlled Rollover
D. NOOS Autonomous Run
```

观察：Decision retention、Constraint violation、Rejected reopen、Repeated discussion、Open Question closure、Useful progress、Human intervention、Page performance、Recovery correctness。

真实 Eval 还应保持相同任务起点、相近 budget，并尽量采用独立 judge / human blind review。

---

# 18. 当前 Acceptance Criteria

- **Continuity**：跨多个 Provider Conversation 后仍是一项连续工作。
- **State Fidelity**：Committed constraint / decision 不因 compaction/rollover 静默变化。
- **Authority Safety**：未授权模型不能修改 Committed State / scope / external source。
- **Negative Memory**：重要 rejection 不无条件复活。
- **Progress**：自主轮次能关闭问题，而不是只增加文本。
- **Performance**：长 Run 不要求用户手工维护卡顿页面。
- **Recovery**：刷新、关闭 tab、重启后能幂等恢复。
- **Human Attention**：只在真正 Authority Boundary 上叫回用户。

---

# 19. NOOS 整体架构：两个平面

```text
NOOS
│
├─ Knowledge / Context Plane
│  ├─ Vault / Crystal / Handoff / Artifact
│  ├─ Reference
│  └─ Context Broker
│
└─ Execution Plane
   └─ Harness Runtime
      ├─ Runtime Object Model
      ├─ Run State
      ├─ SourceRef / EvidenceRef
      ├─ Authority / Promotion Policy
      ├─ State Delta + Reducer
      ├─ Context Compiler
      ├─ Action Policy
      ├─ Session Continuity
      ├─ Execution Journal
      ├─ Review Orchestration
      └─ Provider Adapters
```

此前 NOOS 的主要命题是：

> **NOOS owns user context.**

现在补上的第二个命题是：

> **NOOS owns the continuity of AI work.**

---

# 20. 当前 Design Baseline

目前冻结：

1. Run 是核心 durable object。
2. Provider Conversation 是 replaceable carrier；Browser Session 才是 disposable。
3. 一个 Logical Thread 同时至多一个 current Provider Conversation。
4. Raw Conversation 不是 Current State。
5. Stateful Compaction 优于普通 Summary。
6. Context Store 与 Context Projection 分离。
7. Operational Authority 与 Epistemic Authority 分离。
8. SourceRef 与 EvidenceRef 分离；Source ≠ Claim。
9. Source observation 内容身份 immutable。
10. Committed State 与 Canonical Source 使用不同语义。
11. LLM proposes；Policy authorizes；Reducer applies；NOOS records。
12. Proposal / Authorization / Reducer 在 v0 使用同一事务边界。
13. Human Gate 由 Policy durable 持有，不由 Reducer产生。
14. Refresh 与 Rollover 分开。
15. Recovery 需要 Execution Journal 与 idempotency。
16. Reviewer Orchestration 建立在 Harness 之上。
17. Harness Runtime 属于 NOOS Hub 的 Execution subsystem。
18. 必须通过真实任务 Eval 验证 Harness。

---

# 21. 下一步文档顺序

当前顺序：

1. **Runtime Object Model & Authority Model v0** — Candidate
2. **State Delta + Reducer Contract v0** — Candidate
3. **Continuation State Machine v0** — 待前两篇接口对审通过后进入
4. **Execution Journal & Recovery Contract v0**

`Control Block` 只作为 bootstrap transport，不单独占据架构中心。

---

## Related

- [Runtime Object Model & Authority Model v0](runtime-object-authority-model.md)
- [State Delta + Reducer Contract v0](state-delta-reducer-contract.md)
- [Branding / Naming](../branding/naming.md)
