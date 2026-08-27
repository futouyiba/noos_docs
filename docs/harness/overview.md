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

# 1. 真正要解决的，不是“自动点继续”

典型复杂设计流程更像：

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
检查策略空间
  ↓
……
```

其中大量“继续”并不包含新的用户判断。用户真正需要介入的时刻通常只是：

- 两个方案都合理，需要产品选择；
- 需要改变任务范围；
- 要推翻已经提交的重要决定；
- 要执行外部写入或不可逆动作。

因此，Auto Continue 只是表层需求。真正的问题是：

> **怎样让一项 AI 工作在用户不持续盯着页面的情况下，仍能向前推进、保持状态、处理故障，并在真正需要人类权威时暂停。**

---

# 2. 需求分成四层

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

它们是 Runtime 的输入，不是 Runtime 本身。

---

# 3. 核心对象边界：Run ≠ Conversation ≠ Browser Session

```text
Run = 用户真正拥有的长期工作
Logical Thread = Run 内一个持续角色 / 思考维度
Provider Conversation = 外部平台保存的一段对话载体
Browser Session = 某次网页运行与挂接环境
```

例如：

```text
Run：World Condition → Fish Response
│
├─ Logical Thread：Main Design
│   ├─ Provider Conversation A
│   ├─ Provider Conversation B
│   └─ Provider Conversation C
│
├─ Logical Thread：Domain Review
│   └─ Provider Conversation D
│
└─ Logical Thread：Production Review
    └─ Provider Conversation E

Browser Session / Tab / Adapter Attachment
    ↳ 只是某一时刻挂接其中一个 Conversation 的临时执行表面
```

因此：

- **Run：durable，NOOS-owned。**
- **Logical Thread：durable，NOOS-owned。**
- **Provider Conversation：replaceable carrier，通常 provider-persisted，并承担 provenance。**
- **Browser Session / Adapter Attachment：disposable runtime resource。**

v0 再增加一个简单但重要的 invariant：

> **一个 Logical Thread 同一时刻至多有一个 current Provider Conversation。**

Reviewer fan-out 应建多个 Logical Thread，而不是让一个 Thread 同时拥有多个 current carrier。

详细定义见 [`runtime-object-authority-model.md`](runtime-object-authority-model.md)。

---

# 4. NOOS 拥有的不是“全部聊天记录”，而是结构化工作状态

只保存完整 conversation，不等于拥有上下文。

真正需要的是：

```text
Raw Transcript
      ↓
State Extraction / State Delta
      ↓
Run State
      ↓
Context Compiler
      ↓
Next Execution Projection
```

信息可分三层：

## 4.1 Durable / Committed Context

长期稳定：

- Goal / Deliverable；
- Scope；
- Constraints；
- Committed Decisions；
- Rejected Decisions；
- 重要 Source Ref。

## 4.2 Working Context

当前阶段变化较快：

- Hypotheses；
- Open Questions；
- Current Frontier；
- Active Branch；
- Reviewer Issues；
- 暂时有效的推导状态。

## 4.3 Ephemeral Context

只对最近几轮有意义：

- 某个例子；
- 临时反驳；
- exploratory branch；
- 最近几轮局部 reasoning trace。

原则不是“什么都不能忘”，而是：

> **Raw Conversation 默认只是可回溯证据；只有被明确提升到 State 的内容才获得持续工作语义。**

同时避免 `canonical` 语义重载：

- **Canonical Source**：外部事实的 epistemic authority；
- **Committed State**：Run 内已经正式提交的 constraint / decision / rejection。

---

# 5. Operational Authority 与 Epistemic Authority 必须分开

## Operational Authority

决定：

> 这个 Run 下一步如何执行？

例如 Goal、Scope、Committed Decision、Open Question、Frontier、Next Action。

这些由 Run State + Policy 管理。

## Epistemic Authority

决定：

> 某个外部事实到底应该依据什么？

例如：

- Notion Current Production Fact；
- GitHub 当前代码；
- 用户明确表达的 preference / constraint；
- 外部规范和正式文档。

更合理的链路：

```text
Canonical / Current Source
      ↓
Evidence Snapshot / Source Ref
      ↓
Run State interpretation
```

而不是：

```text
External Fact
↓
模型总结一次
↓
State 永远变成真理
```

Source Ref 不再用一个混合 `authority` enum，而拆成正交维度：

```yaml
origin_kind:
  # user | document | runtime | agent | external

authority_role:
  # canonical | supporting | reference

temporal_status:
  # current | historical | unknown

claim_kind:
  # fact | preference | constraint | decision | inference

version:
observed_at:
freshness:
content_fingerprint:
```

Authority resolver 还必须先看 `claim_kind`：

- 对 preference / goal / scope 等 normative claim，用户明确表达通常高于 agent inference；
- 对当前代码、Production 行为等 factual claim，应依据该领域的 canonical/current source，而不是固定 `user > document`。

一句话：

> **Run State 是 operational working authority，不是所有事实的 ultimate source of truth。**

---

# 6. Compaction 不是 Summary，而是 Stateful Compaction

普通“帮我总结以上对话”很容易把 Committed、Hypothesis、Rejected、Open Question、Evidence 压成一篇流畅但失真的文字。

真正需要：

> **Stateful Compaction：从最近一段轨迹中提取结构化 State Delta，并更新短期 Carry Context。**

Raw Transcript 仍然归档，但退出 active working context。

这让 Conversation Rollover 同时成为一次 Context Garbage Collection：旧 exploratory noise 可以退出下一段显式上下文，而已经提交的状态不会被自然语言摘要悄悄重写。

---

# 7. LLM 不能直接重写 State：Proposal → Policy → Reducer

正确链路：

```text
LLM / Tool / User / Source Observer
   ↓
State Delta Proposal
   ↓
Authority / Promotion Policy
   ↓
Authorized Delta
   ↓
Reducer
   ↓
State vN+1
   ↓
Audit Record
```

长期 invariant：

> **LLM proposes; Policy authorizes; Reducer applies; NOOS records.**

Reducer 负责状态完整性，不负责决定谁有资格确认产品决策。

例如 agent 默认可被授权：

```text
add hypothesis
add evidence
set frontier
close exploratory question
```

通常需要 Human Gate：

```text
product choice
change scope
authorize superseding committed decision
external irreversible write
```

正式 operation / provenance / concurrency / atomicity 见 [`state-delta-reducer-contract.md`](state-delta-reducer-contract.md)。

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

Context Compiler 负责：

1. 选择；
2. 裁剪；
3. 排序；
4. provenance；
5. working-set budget。

这才是 NOOS 真正能影响 Chatbot 工作上下文的地方。

---

# 9. Context Control 的真实边界

NOOS 无法完全控制 ChatGPT / Claude 的 context window，也不能可靠控制：

- system prompt；
- account memory；
- project instruction；
- provider-side summary；
- tool state；
- provider policy。

因此不用“Hard Context Control”。

## Same Conversation：Soft Guidance

NOOS 可以重申 constraint、注入 relevant decision、检测 drift、给出 focused next action，但不能真正删除既有 history。

## New Provider Conversation：Controlled Context Reset

Rollover 后，NOOS 可以更强地决定显式投喂的工作历史：

```text
Harness Contract
+ Goal
+ Committed / Working State
+ Relevant Source Evidence
+ Carry Context
+ Next Action
```

所以：

> **Rollover 的价值不只是性能，而是获得更清晰的 explicit-history boundary。**

---

# 10. Session Continuity：Refresh 与 Rollover 是两类手段

## Safe Refresh

重建浏览器运行环境。Provider Conversation 不变。

## Conversation Rollover

重建模型工作的显式上下文边界。Provider Conversation 更换，但 Logical Thread / Run 不变。

因此：

```text
Performance degradation
→ Prefer Refresh

Context / semantic pressure
→ Compact + Rollover
```

Round Count 只能是 signal，不能成为“每 20 轮换房间”的死规则。

页面卡顿的具体机理目前仍是 Candidate Mechanism：

> **instrument first, optimize second.**

---

# 11. Continuation 不是“继续”，而是 Next Action Policy

v0 动作保持很小：

```text
CONTINUE_FOCUSED
COMPACT
REFRESH
ROLLOVER
ASK_HUMAN
COMPLETE
```

决策优先级建议：

```text
1. User Activity
2. State Integrity
3. Authority Boundary
4. Completion
5. Page Health
6. Context Health
7. Progress
```

“继续”应尽量被具体化为 focused next action，而不是让模型和自己无限互发“继续”。

---

# 12. Recovery 不能只靠 Checkpoint：还需要 Execution Journal

只有 `state_version` 和 `checkpoint_id` 不足以恢复：系统还必须知道上一条 action 是否已发送、provider 是否已观察、结果是否已返回、State Proposal 是否被接受。

因此：

```text
State Store       = 现在是什么
Execution Journal = 刚才发生了什么
```

但这里不冻结一个简单的 `planned → sent → observed → committed` 单轴状态机，因为以下是不同维度：

```text
Provider side effect happened
Assistant result observed
State proposal accepted/rejected
```

详细 dispatch / reconciliation / state application / idempotency 留给《Execution Journal & Recovery Contract v0》。

---

# 13. Harness Control Block 只是 Bootstrap Transport

MVP 可以让 Worker 在输出末尾追加机器可读 Control Proposal，以避免另起 controller model。

但 architectural contract 是：

> **Control Proposal**

HTML marker 只是一种 transport。未来可以来自 structured output、独立 controller、local evaluator、Shadow Controller 等。

---

# 14. Multi-Conversation Review 建立在 Harness 之上

```text
Run State v37
      ↓
Review Snapshot RS-008
      ↓
Domain / Product / Runtime Reviewer
      ↓
Structured Review Issues
      ↓
Owner Adjudication
      ↓
State Delta
```

Review Issue 必须带：

```yaml
review_snapshot_id:
base_state_version:
```

这样 Main State 推进后才能识别 stale review，而不是把旧审查意见直接写进当前状态。

---

# 15. Harness Runtime 属于 NOOS Hub

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

除非真实隔离/性能需求证明需要拆服务，否则不要再制造第二个本地 authority center。

---

# 16. MVP 拆成五个可验证里程碑

## M0 — Run Continuity Proof

```text
Take Over
→ Create Run
→ Save Checkpoint
→ Refresh / Close Browser
→ Resume
```

验证：工作能否从网页生命周期中解耦。

## M1 — Controlled Rollover

```text
Conversation A
→ Stateful Compaction
→ Context Projection
→ Conversation B
→ Same Run
```

验证：External State + Compaction + Rollover 本身是否创造价值。

## M2 — Autonomous Continuation

加入：

```text
CONTINUE_FOCUSED
ASK_HUMAN
COMPLETE
```

验证：用户不持续盯页面时是否仍有 substantive progress。

## M3 — Performance Self-Healing

加入 telemetry、safe refresh、auto reattach、refresh/rollover pressure。

验证：页面性能问题能否变成 Runtime 自处理故障。

## M4 — Reviewer Orchestration

加入 review snapshot、orthogonal reviewer、issue merge、owner adjudication。

验证：purpose-built reviewer projection 是否优于一个超长主 Chat 自我审查。

---

# 17. Eval：拆开机制贡献

实验分组：

```text
A. Long Chat
   人工 Continue，不 Rollover

B. Human-managed Rollover
   人写 checkpoint，手工开新 Chat

C. NOOS Controlled Rollover
   Stateful Compaction + Projection + Rollover

D. NOOS Autonomous Run
   C + Action Policy
```

观察：

- Decision retention；
- Constraint violation；
- Rejected option reopen rate；
- Repeated discussion；
- Open Question closure；
- Useful progress / round；
- Human intervention；
- Page performance；
- Recovery correctness。

真正实验时应尽量保持：

```text
相同任务起点
相近 token / wall-clock budget
独立 judge 或 human blind review
```

避免让执行 Harness 的同一个模型自己给 Harness 打分。

同时保持严谨：长上下文研究只能提供 plausible mechanism；具体到 ChatGPT 网页长聊的重复、漂移、旧决策复活，仍需 NOOS 自己的真实任务验证。

---

# 18. Acceptance Criteria

### Continuity

跨多个 Provider Conversation 后，用户仍认为是同一个 Run。

### State Fidelity

Committed constraint / decision 不因 compaction 或 rollover 静默变化。

### Authority Safety

模型不能无授权把 hypothesis 升级为 committed decision，或自行改变 scope / 执行外部不可逆写入。

### Negative Memory

已 rejected 的重要方案不会无条件复活。

### Progress

自主轮次能够关闭问题，而不是只增加文字。

### Performance

长 Run 不要求用户因为页面卡顿而手工重开工作。

### Recovery

刷新、关闭 tab、浏览器重启后可通过 Checkpoint + Execution Journal 幂等恢复。

### Human Attention

用户主要在真正 Authority Boundary 上被叫回来。

---

# 19. NOOS 因此形成两个平面

```text
NOOS
│
├─ Knowledge / Context Plane
│  ├─ Vault
│  ├─ Crystal
│  ├─ Handoff
│  ├─ Artifact
│  ├─ Reference
│  └─ Context Broker
│
└─ Execution Plane
   └─ Harness Runtime
      ├─ Runtime Object Model
      ├─ Run State
      ├─ Authority / Promotion Policy
      ├─ State Delta + Reducer
      ├─ Context Compiler
      ├─ Action Policy
      ├─ Session Continuity
      ├─ Execution Journal
      ├─ Review Orchestration
      └─ Provider Adapters
```

此前 NOOS 的主要命题：

> **NOOS owns user context.**

现在补上的第二个命题：

> **NOOS owns the continuity of AI work.**

这是对原 Context Hub 的补全，不是战略转向。

---

# 20. 当前 Design Baseline

目前冻结：

1. **Run 是核心 durable object。**
2. **Provider Conversation 是 replaceable carrier，不是 disposable Browser Session。**
3. **一个 Logical Thread v0 同时至多一个 current Provider Conversation。**
4. **Raw Conversation 不是 Current State。**
5. **Committed State 与 Canonical Source 分词。**
6. **Stateful Compaction 优于普通 Summary。**
7. **Context Store 与 Context Projection 分离。**
8. **Operational Authority 与 Epistemic Authority 分离。**
9. **Source Ref 的 origin / authority role / temporal status / claim kind 正交拆分。**
10. **LLM proposes；Policy authorizes；Reducer applies；NOOS records。**
11. **Refresh 与 Rollover 分开。**
12. **Recovery 需要 Execution Journal 与 idempotency。**
13. **Reviewer Orchestration 建立在 Harness 之上。**
14. **Harness Runtime 属于 NOOS Hub 的 Execution subsystem。**
15. **必须通过真实任务 Eval 验证。**

---

# 21. 下一步 Contract 顺序

现在停止继续打磨 Overview，用实现级 Contract 和 Eval 逼出真实问题。

1. **Runtime Object Model & Authority Model v0** — Design Candidate  
   [`runtime-object-authority-model.md`](runtime-object-authority-model.md)

2. **State Delta + Reducer Contract v0** — Design Candidate  
   [`state-delta-reducer-contract.md`](state-delta-reducer-contract.md)

3. **Continuation State Machine v0** — Next  
   Continue / Human Gate / Complete / Compact / Rollover / Refresh。

4. **Execution Journal & Recovery Contract v0**  
   Dispatch / observed result / reconciliation / state application / idempotency。

`Control Block` 只作为 Continuation Runtime 的 bootstrap transport，不单独占据架构中心。

---

## Related

- [Runtime Object Model & Authority Model v0](runtime-object-authority-model.md)
- [State Delta + Reducer Contract v0](state-delta-reducer-contract.md)
- [Branding / Naming](../branding/naming.md)
