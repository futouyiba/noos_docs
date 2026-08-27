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

旁边还有一个独立平面：

```text
Vault / Wiki / Notion / Git / Files / Reference
```

它们是 Context Source，不是 Runtime 本身。

这一区分很重要。否则 NOOS 很容易再次膨胀成一个“大而全 AI 平台”。

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

这一区分直接决定后面的 Recovery、Rollover、provenance 和 adapter API。

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

可以把 Run 内的信息分成三层：

## 4.1 Durable Context

长期稳定：

- Goal / Deliverable；
- Scope；
- Constraints；
- Confirmed Decisions；
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

> **默认 Raw Conversation 只是可回溯证据；只有被明确提升到 State 的内容才获得持续的工作语义。**

---

# 5. 但必须区分 Operational Authority 与 Epistemic Authority

“Current State 是下一轮权威输入”这句话只能用于**运行控制和本 Run 的内部决策状态**，不能扩大成“State 是所有事实的最终真理”。

必须区分两种 Authority：

## Operational Authority

决定：

> 这个 Run 下一步如何执行？

例如：

- 当前 Goal；
- Scope；
- 已确认的 Run Decision；
- Open Question；
- Frontier；
- Next Action。

这些由 Run State 管理。

## Epistemic Authority

决定：

> 某个外部事实到底是真的什么？

例如：

- Notion Current Production Fact；
- GitHub 当前代码；
- 用户明确输入的 constraint；
- 外部规范和正式文档。

这些不能因为被模型总结进 State，就失去原始 Source Authority。

更合理的链路是：

```text
Canonical Source
      ↓
Evidence Snapshot / Source Ref
      ↓
Run State interpretation
```

而不是：

```text
Canonical Source
      ↓
模型总结一次
      ↓
State 永远变成真理
```

`source_ref` 因此应逐步允许表达：

```yaml
uri:
authority:
version:
observed_at:
freshness:
```

一句话：

> **Run State 是 working authority，不是所有事实的 ultimate source of truth。**

---

# 6. Compaction 不是 Summary，而是 Stateful Compaction

如果每二十轮都让模型：

> “请总结以上对话。”

它很容易把 Confirmed、Hypothesis、Rejected、Open Question、Evidence 压成一篇流畅但失真的文字。

真正需要的是：

> **Stateful Compaction：从最近一段轨迹中提取结构化 State Delta，并更新短期 Carry Context。**

例如：

```text
CONFIRMED
D-017 Feeding Motivation 当前不作为 canonical Core State。

RATIONALE
目前不存在足够独立的生命周期或 producer-consumer ownership。

REOPEN CONDITION
未来若出现独立持久化需求，则重新评估。

OPEN
Runtime 中应该用什么 derived representation 表达？
```

Raw Transcript 仍然归档，但退出 active working context。

---

# 7. LLM 不能直接重写 State：Proposal → Policy → Reducer

仅仅有 Reducer 还不够。

Reducer 能保证 schema 和状态一致性，却不能回答：

> “这个模型有没有资格把一个 hypothesis 直接升级为 Confirmed Decision？”

因此正确链路应该是：

```text
LLM / Tool
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

可以把原则写成：

> **LLM proposes; Policy authorizes; Reducer applies; NOOS records.**

例如默认可自动执行：

```yaml
agent_may:
  - add_hypothesis
  - add_evidence
  - set_frontier
  - close_exploratory_question
```

而通常需要 Human Gate：

```yaml
human_required:
  - product_choice
  - change_scope
  - supersede_confirmed_decision
  - external_write
```

这样 Human Gate 才真正和 State Transition 连起来。

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

一个 Run 可能关联几十万 token 的资料，但当前一次执行可能只需要：

```text
Task Contract
Relevant Constraints
Relevant Decisions
Relevant Rejected
Current Frontier
Working Set
Relevant Source Excerpts
Next Action
```

Context Compiler 的价值不在“拼接资料”，而在：

1. **选择**；
2. **裁剪**；
3. **排序**；
4. **维护 provenance**；
5. **控制 working-set 大小**。

这才是 NOOS 真正能影响 Chatbot 工作上下文的地方。

---

# 9. Context Control 的真实边界

NOOS 无法完全控制 ChatGPT / Claude 的 context window。

它通常不能可靠读取或控制：

- system prompt；
- account memory；
- project instruction；
- provider-side summary；
- tool state；
- provider policy。

因此不再使用“Hard Context Control”这种过强表述。

更准确地说：

## Same Conversation：Soft Guidance

NOOS 可以：

- 重申 constraint；
- 注入 relevant decision；
- 检测 drift；
- 给出 focused next action。

但无法真正删除既有 conversation history。

## New Provider Conversation：Controlled Context Reset

Rollover 后，NOOS 可以更强地决定**显式投喂的工作历史**：

```text
Harness Contract
+ Goal
+ Active State
+ Relevant Source Evidence
+ Carry Context
+ Next Action
```

因此：

> **Rollover 的价值不只是性能，而是获得更清晰的 explicit-history boundary。**

---

# 10. Session Continuity：Refresh 与 Rollover 是两类不同治疗手段

## Safe Refresh

目标：重建浏览器运行环境。

适合处理：

- 当前页面明显变慢；
- DOM / streaming 生命周期积累；
- adapter attachment 状态异常。

Provider Conversation 不变。

## Conversation Rollover

目标：重建模型工作的显式上下文边界。

适合处理：

- conversation history 太长；
- semantic phase 已切换；
- working set 已变化；
- 重复、漂移、旧 decision 复活增多。

Provider Conversation 更换，但仍属于同一个 Logical Thread / Run。

因此：

```text
Performance degradation
→ Prefer Refresh

Context / semantic pressure
→ Compact + Rollover
```

Round Count 只能是 signal，不能变成“每 20 轮强制换房间”的死规则。

页面卡顿的具体机理目前也只应视为 Candidate Mechanism。MVP 应遵循：

> **instrument first, optimize second.**

---

# 11. Continuation 不是“继续”，而是 Next Action Policy

Harness 不应该无脑发送：

> 继续。

它真正需要决定的是：

> 下一步动作是什么？

v0 动作可以保持很小：

```text
CONTINUE_FOCUSED
COMPACT
REFRESH
ROLLOVER
ASK_HUMAN
COMPLETE
```

`CONTINUE_FOCUSED` 更像：

> 继续处理 Q-014；不要重新总结已关闭模型，重点检查这一拆分是否产生 double-count。

这比裸“继续”更容易产生 substantive progress。

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

只有 `state_version` 和 `checkpoint_id` 不足以可靠恢复。

典型故障：

```text
NOOS 点击 Send
↓
Provider 已收到 prompt
↓
Browser 崩溃
↓
NOOS 重启
```

系统必须知道上一条 Action 到底执行到了哪一步，否则可能重复发送。

因此需要一个最小 Execution Journal：

```yaml
action_id:
run_id:
logical_thread_id:
provider_conversation_id:
base_state_version:
action:
status: planned | sent | observed | committed
provider_message_ref:
created_at:
```

可以把它理解成：

```text
State Store = 现在是什么
Execution Journal = 刚才发生了什么
```

Recovery 的目标不是“重新点一次按钮”，而是**幂等地恢复到可证明的执行状态**。

---

# 13. Harness Control Block 只是 Bootstrap Transport

MVP 阶段，为了避免额外 controller model，可以让 Worker 在正常输出末尾追加一个机器可读的 Control Proposal：

```html
<!-- NOOS:CONTROL:BEGIN -->
{
  "base_state_version": 17,
  "progress": "advanced",
  "next_action": "continue_focused",
  "state_delta": []
}
<!-- NOOS:CONTROL:END -->
```

这是一个很实用的 bootstrap 方案。

但架构 contract 应该叫：

> **Control Proposal**

HTML marker 只是 transport implementation 之一。

未来同一个 contract 可以来自：

- response marker；
- provider structured output；
- 独立 controller call；
- local evaluator；
- Shadow Controller。

不能把架构绑死在“ChatGPT 必须在结尾输出 JSON”。

---

# 14. Multi-Conversation Review：建立在 Harness 之上

没有 Harness 时：

```text
Main
Reviewer A
Reviewer B
Reviewer C
```

只是制造了四条更难管理的聊天。

有 Harness 后：

```text
Run State v37
      ↓
Review Snapshot RS-008
      ↓
┌──────────────┬──────────────┬──────────────┐
Domain Review  Product Review Runtime Review
└──────────────┴──────────────┴──────────────┘
      ↓
Structured Review Issues
      ↓
Owner Adjudication
      ↓
State Delta
```

Review Issue 至少应该记录：

```yaml
issue_id:
base_state_version:
review_snapshot_id:
dimension:
severity:
target:
claim:
evidence:
suggested_action:
```

`base_state_version` 很关键，因为 Reviewer 返回时 Main State 可能已经推进；Harness 必须能识别 stale review。

这一层属于后续阶段，不是最初 MVP 的阻塞项。

---

# 15. Harness 与 NOOS Hub 的关系

NOOS 不应该再制造第二个本地中枢。

第一阶段直接定义：

```text
NOOS Hub
├─ Vault / Artifact Store
├─ Context Broker
├─ Harness Runtime
└─ Tool Router

Browser Shuttle
└─ Provider / Chatbot Adapter
```

因此：

> **Harness Runtime 是 Hub 的 Execution subsystem；Shuttle 是浏览器侧执行与连接层。**

除非以后性能或隔离要求证明必须拆服务，否则不要提前制造新的 daemon。

这与 NOOS 既有的 Context Hub / Shuttle 方向保持一致。

---

# 16. MVP 不再叫一个巨大 v0，而拆成五个可验证里程碑

## M0 — Run Continuity Proof

只验证：

```text
Take Over
→ Create Run
→ Save Checkpoint
→ Refresh / Close Browser
→ Resume
```

核心问题：

> **工作能否从网页生命周期中解耦？**

## M1 — Controlled Rollover

验证：

```text
Conversation A
→ Stateful Compaction
→ Context Projection
→ Conversation B
→ Same Run
```

核心问题：

> **换 Conversation 后，用户是否仍认为这是同一项连续工作，而且状态没有明显损失？**

## M2 — Autonomous Continuation

加入：

```text
CONTINUE_FOCUSED
ASK_HUMAN
COMPLETE
```

核心问题：

> **用户是否可以不持续盯着页面，而工作仍产生有效进展？**

## M3 — Performance Self-Healing

再加入：

```text
performance telemetry
safe refresh
auto reattach
refresh / rollover pressure
```

核心问题：

> **页面性能问题能否变成 Runtime 自己处理的故障，而不是用户手工维护？**

## M4 — Reviewer Orchestration

最后加入：

```text
review snapshot
orthogonal reviewer
issue merge
owner adjudication
```

核心问题：

> **purpose-built reviewer projection 是否比一个超长主 Chat 自我审查更可靠？**

---

# 17. Eval：必须拆开机制贡献，而不是只做 Harness vs Long Chat

整体 A/B 只能回答“Harness 有没有用”，不能回答“到底什么有用”。

更好的实验分组：

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

主要观察：

- Decision retention；
- Constraint violation；
- Rejected option reopen rate；
- Repeated discussion；
- Open Question closure；
- Useful progress / round；
- Human intervention count；
- Page performance；
- Recovery correctness。

需要保持一个严谨表述：

> **我们观察到某些长 conversation 会出现重复、漂移和旧决策复活；长上下文研究为此提供 plausible mechanism，但具体到 ChatGPT 网页长聊是否由同一机制导致，仍然需要 NOOS 自己的真实任务实验验证。**

---

# 18. 当前 Acceptance Criteria

Harness 的验收不能是“它会自动点继续”。

至少应该有：

### Continuity

Run 跨多个 Provider Conversation 后，用户仍认为它是一项连续工作。

### State Fidelity

已确认 constraint / decision 不因 compaction 或 rollover 静默变化。

### Authority Safety

模型不能在没有授权的情况下，把 hypothesis 升级为 confirmed decision，或执行 scope / external write 变化。

### Negative Memory

已 rejected 的重要方案不会无条件复活。

### Progress

自主轮次能够关闭问题，而不是只增加文字量。

### Performance

长 Run 不要求用户因为页面卡顿而手工重开工作。

### Recovery

刷新、关闭 tab、浏览器重启后，能通过 Checkpoint + Execution Journal 幂等恢复。

### Human Attention

用户只在真正 Authority Boundary 上被叫回来。

---

# 19. NOOS 整体架构因此变成两个平面

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

此前 NOOS 的主要命题是：

> **NOOS owns user context.**

现在补上的第二个命题是：

> **NOOS owns the continuity of AI work.**

这不是战略转向，而是把原本的 Context Hub 补成了可运行的工作系统。

---

# 20. 当前 Design Baseline

目前最值得冻结的结论：

1. **Run 是核心 durable object。**
2. **Provider Conversation 是 replaceable carrier，不是 disposable Browser Session。**
3. **Raw Conversation 不是 Current State。**
4. **Stateful Compaction 优于普通 Summary。**
5. **Context Store 与 Context Projection 必须分离。**
6. **Operational Authority 与 Epistemic Authority 必须分离。**
7. **LLM 只能 propose；Policy authorizes；Reducer applies；NOOS records。**
8. **Refresh 与 Rollover 必须分开。**
9. **Recovery 需要 Execution Journal 与 idempotency。**
10. **Reviewer Orchestration 建立在 Harness 基础之上。**
11. **Harness Runtime 属于 NOOS Hub 的 Execution subsystem。**
12. **必须通过真实任务 Eval 验证，不凭直觉冻结 Harness。**

---

# 21. 下一步文档顺序

接下来不继续横向加功能。

底层 contract 按以下顺序推进：

1. **Runtime Object Model & Authority Model v0**  
   Run / Logical Thread / Provider Conversation / Browser Session / Turn / Checkpoint / Authority / Promotion。

2. **State Delta + Reducer Contract v0**  
   State schema / operations / invariants / provenance / authorization boundary。

3. **Continuation State Machine v0**  
   Continue / Human Gate / Complete / Compact / Rollover / Refresh。

4. **Execution Journal & Recovery Contract v0**  
   planned / sent / observed / committed / idempotency / reconciliation。

`Control Block` 只作为 Continuation Runtime 的 bootstrap transport，不单独占据架构中心。

---

## Related

- [Runtime Object Model & Authority Model v0](runtime-object-authority-model.md)
- [Branding / Naming](../branding/naming.md)
