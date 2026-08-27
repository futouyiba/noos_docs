# NOOS Harness：把 Chatbot 从长对话变成可持续运行的 AI 工作执行器

> **状态**：Product Architecture Candidate v0.1  
> **日期**：2026-08-27  
> **项目**：NOOS  
> **候选中文名**：怒思（当前首选候选）  
> **主题**：Browser Chatbot Harness / Session Continuity / Context Control / Multi-Conversation Review

---

## 0. 一句话说明

今天使用 ChatGPT、Claude 这类 Chatbot 做复杂设计时，真正限制工作的往往已经不是“模型不会回答”，而是：**工作无法被持续、稳定、可控地运行。**

一个复杂设计专题可能需要连续推进几十轮；用户需要不断手工发送“继续”；对话越来越长以后网页开始卡顿；模型也会逐渐重复、漂移、忘记已关闭的问题；如果再开几个独立对话做领域审查、产品审查、实现审查，用户还要人工在这些对话之间搬运上下文和意见。

NOOS Harness 要解决的不是“再造一个 ChatGPT”，而是把这些原本靠用户手工维持的工作，变成一个由 NOOS 管理的长期 Run：

> **NOOS 持有任务状态和显式工作上下文；Chatbot 负责阶段性推理。Conversation 可以刷新、压缩、替换和分叉，而 Run 始终连续。**

---

# 1. 我们真正遇到的问题是什么？

先看一个很普通、但很典型的复杂设计过程。

用户提出一个问题：

> “设计 World Condition → Fish Response 的产品因果模型。”

第一轮回答通常不会直接达到稳定设计。真实过程更像这样：

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
审查概念 ownership
  ↓
继续
  ↓
发现 double-count
  ↓
继续
  ↓
检查玩家策略空间
  ↓
……
```

这个过程持续二三十轮并不罕见。

问题在于，其中大量“继续”并不包含新的用户判断。用户真正提供价值的时刻，通常只是少数几个：

- 两个方案都合理，需要做产品选择；
- 要改变讨论范围；
- 要推翻已经确认的重要决定；
- 要向正式文档、代码或外部系统执行不可逆写入。

剩下的大量工作，其实可以由一个 Harness 自动推进。

于是第一个需求出现了：

> **不要让我守着 ChatGPT 点几十次“继续”。**

但只做自动继续，很快又会暴露第二个问题。

---

# 2. 为什么“自动发送继续”远远不够？

如果插件只是：

```text
回答结束
→ 自动发送“继续”
→ 回答结束
→ 自动发送“继续”
```

它最多是一个自动点击器。

而真正长期运行以后，会出现至少四类问题。

## 2.1 页面越来越卡

长时间 streaming、大量 DOM、代码块、附件、React 状态以及插件自身 observer 都可能让 ChatGPT 页面越来越重。

有时刷新网页就能恢复相当一部分性能。

所以需要：

> **Safe Refresh：在确认当前回答已经结束、用户没有未提交输入、Run 状态已经持久化后，自动刷新并恢复。**

## 2.2 Conversation 本身越来越长

即使网页刷新以后恢复流畅，一个拥有上百条消息的 conversation 本身仍然越来越重。

更重要的是，问题不仅是性能。

长对话还会出现：

- 旧假设与新结论混在一起；
- 已经 rejected 的方案重新出现；
- 已关闭的问题被重新打开；
- 术语慢慢漂移；
- 模型花越来越多精力理解历史，而不是解决当前问题。

这可以称为 **Context Rot**。

因此需要：

> **Conversation Rollover：把当前工作状态压缩成受控上下文，开启一个新的 Chatbot conversation，并继续同一个逻辑任务。**

## 2.3 多个审查对话开始失控

复杂设计往往不能只靠主线程自我审查。

例如可能需要：

```text
Main Design
├─ Domain Reviewer
├─ Product / Strategy Reviewer
├─ Architecture Reviewer
└─ Production Reality Reviewer
```

今天这些对话之间的上下文搬运、review 结果回收、冲突整理、修改主方案，几乎全部靠人手完成。

因此需要：

> **Multi-Conversation Orchestration：让不同 conversation 扮演不同正交角色，并由 Harness 负责分发、回收与裁决流程。**

## 2.4 ChatGPT 的上下文并不真正属于用户

在网页 ChatGPT 中，用户无法准确控制或观察服务端究竟怎样管理：

- system context；
- memory；
- project instructions；
- 历史消息截断；
- 内部 summary；
- tool state。

因此 NOOS 不应该承诺“完全控制 ChatGPT 的 context window”。

但它可以做到一个更现实、也已经非常有价值的事情：

> **把任务真正依赖的 authoritative explicit context 放在 NOOS 中维护，并尽量减少工作对 ChatGPT 隐式聊天历史的依赖。**

这就是 Harness 的核心。

---

# 3. 最重要的抽象：Run 不是 Conversation

传统 Chatbot 的默认心智模型是：

```text
Conversation = 这项工作
```

NOOS Harness 应改成：

```text
Run = 这项工作
Conversation = 这项工作的一次执行载体
```

例如：

```text
Run：World Condition → Fish Response
│
├─ Logical Thread：Main Design
│   ├─ ChatGPT Conversation A
│   ├─ ChatGPT Conversation B
│   └─ ChatGPT Conversation C
│
├─ Logical Thread：Domain Review
│   ├─ ChatGPT Conversation D
│   └─ ChatGPT Conversation E
│
└─ Logical Thread：Production Review
    └─ ChatGPT Conversation F
```

用户看到的始终是一个 Run。

底下究竟刷新了几次网页、换了几个 ChatGPT conversation，应该只是实现细节。

因此可以形成一个非常重要的设计原则：

> **Run 是 durable object；Execution Session 是 disposable runtime resource。**

这意味着：

- 刷新页面不能丢工作；
- ChatGPT tab 被关闭不能丢工作；
- Conversation 太长可以换；
- 浏览器重启以后可以恢复；
- 将来甚至可以从 ChatGPT 切换到 Claude，而 Run 仍然存在。

---

# 4. NOOS Harness 到底“拥有”什么？

一个容易走错的方向是：把所有聊天记录存下来，然后认为自己已经拥有上下文。

这还不够。

Harness 真正应该拥有的是 **Current State**。

原始聊天只是证据和轨迹；当前状态才是下一轮工作的权威输入。

可以把信息分成三层。

## 4.1 Durable Context

长期必须稳定存在：

- Goal；
- Deliverable；
- Scope；
- Constraints；
- Confirmed Decisions；
- Rejected Decisions；
- 重要来源与事实。

## 4.2 Working Context

当前阶段还在变化：

- Hypotheses；
- Open Questions；
- Current Frontier；
- Active Branch；
- 当前比较的方案；
- 最近 reviewer 发现的问题。

## 4.3 Ephemeral Context

只在短时间内有价值：

- 某个例子；
- 临时反驳；
- exploratory branch；
- 最近几轮的局部推理。

这类信息应该允许快速淘汰。

因此 Harness 的原则不是“什么都不能忘”，而是：

> **默认原始 conversation history 都是可退出 working context 的；只有被提升为 State 的内容才获得持久语义。**

---

# 5. Run State v0

第一版不需要一上来建立复杂知识图谱。

一个最小 Run State 可以是：

```yaml
run:
  id:
  title:
  goal:
  deliverable:

  scope:
    in: []
    out: []

  phase:
  status:

state:
  constraints: []
  decisions: []
  rejected: []

  hypotheses: []
  open_questions: []

  frontier:
    current_problem:
    current_position:
    next_intent:

  working_set:
    active_branch:
    recent_reasoning:
    unresolved_evidence:

  source_refs: []

runtime:
  primary_thread:
  state_version:
  checkpoint_id:
```

其中最值得强调的是四类信息。

## 5.1 Decisions

已经明确确认的决定。

一个 Decision 不应该只有一句结论，还应该知道：

```yaml
id: D-017
claim:
rationale:
status: active
authority:
reopen_if:
origin:
```

`reopen_if` 很重要。

它让“以后是否允许重新讨论”从模糊感觉变成显式条件。

## 5.2 Rejected

被否掉的重要方案必须单独保存。

这是 **Negative Memory**。

因为 LLM 很容易在 20 轮以后重新提出一个此前已经被否掉的“自然方案”。

## 5.3 Open Questions

长期任务最容易丢失的未必是答案，而是：

> “我们还有什么没有想清楚？”

Open Questions 应该是一等对象，而不是散落在聊天文本中。

## 5.4 Frontier

`frontier` 表示：

> **现在究竟推到哪里，下一步为什么继续。**

例如：

```yaml
frontier:
  current_problem: Fish-specific interpretation 应产生哪些中间语义？
  current_position: 已排除直接 World Condition → Spawn Weight。
  next_intent: 推导 Spatial / Readiness / Interaction 三条输出路径。
```

有了 Frontier，“继续”才真正具有语义。

---

# 6. Compaction 不是“帮我总结一下”

这是整个 Harness 最容易做错的一块。

如果每 20 轮就让模型：

> “请总结以上对话。”

那么它很容易把：

- Confirmed；
- Working Hypothesis；
- Rejected；
- Open Question；
- Evidence；

全部压成一篇流畅文字。

流畅，但失真。

真正需要的是：

> **Stateful Compaction，而不是 Summarization。**

例如原始讨论可能有 18,000 tokens，压缩后不应该只是：

> “我们讨论了 Feeding Motivation，并认为……”

而应该形成类似：

```text
CONFIRMED
D-017 Feeding Motivation 当前不作为 canonical Core State。

RATIONALE
目前不存在足够独立的生命周期或 producer-consumer ownership。

REOPEN CONDITION
未来若出现跨 tick 独立持久化需求，则重新评估。

OPEN
Runtime 中应该用什么 derived representation 表达？
```

这才能可靠地交给下一段推理。

---

# 7. 模型不能直接重写 State：State Delta + Reducer

为了防止一次 compaction 把旧结论悄悄改掉，模型不应该拥有直接覆盖 State 的权限。

更稳的结构是：

```text
Old State v17
      +
State Delta
      ↓
NOOS Reducer
      ↓
State v18
```

模型只提出 Delta：

```yaml
base_version: 17

operations:
  - op: add_decision
    id: D-023
    claim: ...
    rationale: ...

  - op: close_question
    id: Q-011
    resolution: D-023

  - op: open_question
    id: Q-014
    question: ...

  - op: set_frontier
    current_problem: ...
    next_intent: ...
```

最终修改 State 的是 NOOS Reducer，而不是 ChatGPT。

Reducer 至少应该维护这些 invariant：

1. Decision 不能被静默删除，只能 supersede / reopen；
2. Constraint 不能被模型顺手改写；
3. Rejected 不能无理由复活；
4. Canonical State 变更必须带 provenance；
5. 不确定的内容只能进入 Working State，不能自动升级成 Confirmed。

这形成一个核心安全原则：

> **LLM proposes；NOOS owns state transitions。**

---

# 8. Context Compiler：真正“把控上下文”的地方

NOOS 能存多少资料，并不等于应该给模型多少资料。

假设未来 Vault 中有 60 万 tokens，当前 Run State 有 2 万 tokens，而这一轮真正需要的只有：

```text
Task Contract
+ Relevant Constraints
+ Relevant Decisions
+ Relevant Rejected
+ Open Questions
+ Current Frontier
+ Working Set
+ Relevant Sources
+ Next Action
```

因此 NOOS 需要一个 Context Compiler：

```text
Vault / Sources
      +
Run State
      +
Current Action
      ↓
Context Compiler
      ↓
Context Projection
      ↓
Execution Session
```

注意两个概念的区别：

- **Context Store**：NOOS 知道什么；
- **Context Projection**：这一轮 Chatbot 应该看到什么。

这才是 Harness 意义上的 context control。

它不是控制 ChatGPT 隐藏 system context，而是：

> **控制工作真正依赖的显式上下文，并把历史噪音排除出去。**

---

# 9. Refresh 与 Rollover 不是同一件事

两者都能让“长时间工作继续下去”，但治疗的是不同问题。

## 9.1 Refresh：重建网页运行环境

解决：

- DOM / React 状态积累；
- streaming UI 长时间运行；
- 页面临时变重；
- 插件监听器长期运行。

Conversation 不变。

刷新必须只发生在 Safe Refresh Window：

```text
Assistant 不在 streaming
AND
用户没有未提交输入
AND
最后一条消息已经稳定
AND
Run 状态已经持久化
AND
没有关键写操作正在进行
```

## 9.2 Rollover：重建模型工作环境

解决：

- Conversation 太长；
- Context Rot；
- Semantic Phase 已经切换；
- Working Set 已经完全变化；
- 已出现重复讨论 / 决策复活 / 漂移。

流程是：

```text
Current Conversation
      ↓
State Delta + Carry Context
      ↓
Checkpoint
      ↓
Context Compiler
      ↓
New Conversation
      ↓
Resume Same Run
```

因此可以记住一句非常清楚的区别：

> **Refresh 解决页面运行环境；Rollover 解决模型工作环境。**

---

# 10. 为什么 Rollover 是“硬上下文控制”的关键？

在同一个 ChatGPT conversation 中，NOOS 可以不断提醒模型：

> “不要重新讨论 D-017。”

但旧历史仍然存在。

这是 **Soft Context Control**。

只有进入新 Conversation 后：

```text
Conversation A
      ↓
Compact
      ↓
Compile Projection
      ↓
Conversation B
```

NOOS 才能真正决定：

- 哪些旧内容带进去；
- 哪些彻底不再进入 active context；
- 哪些 rejected 需要重新提醒；
- 哪些 source excerpt 与当前 frontier 有关。

这是 **Hard Context Control**。

因此 Rollover 不是单纯性能优化，而是 Harness 架构的一部分。

---

# 11. “自动继续”应升级成 Action Policy

真正的 Harness 不应该永远发送：

> “继续。”

它应该判断：

> **下一步动作是什么？**

v0 可以只保留少量动作：

```text
CONTINUE_FOCUSED
COMPACT
REFRESH
ROLLOVER
ASK_HUMAN
COMPLETE
```

例如正常推进时，不发送裸“继续”，而是生成：

> “继续处理 Q-014；不要重新总结已有模型，重点检查当前拆分是否造成 cause double-count。如果没有 substantive progress，明确报告已经收敛。”

这会显著减少 LLM 自己和自己无限写同义文字。

---

# 12. Human Gate：什么时候必须叫用户回来？

自动化的价值来自“不需要人一直守着”。

因此不能模型一有疑问就停下来问人。

Human Gate 应该主要保护 **Authority Boundary**。

v0 只需要四类：

## 12.1 Product Choice

多个方案都合理，最终取决于产品偏好。

## 12.2 Scope Change

讨论准备从原任务扩展到另一个系统。

## 12.3 Canonical Change

准备推翻已经确认的重要 decision / constraint。

## 12.4 Irreversible External Action

例如：

- 修改正式 Notion Current；
- 改代码；
- 删除数据；
- 对外发布。

至于：

- 局部证据不足；
- 某一轮没想清楚；
- 需要再审查；
- 可以自己再推一轮；

Harness 应优先自行处理。

---

# 13. Session Continuity Runtime

从用户角度看，不应该暴露“自动刷新”“自动重连”这样的一堆技术功能。

真正的产品能力只有一句：

> **这项工作可以持续运行。**

底下由 Session Continuity Runtime 负责：

```text
Auto Continue / Next Action
        +
Safe Refresh
        +
Crash / Tab Recovery
        +
Compaction
        +
Conversation Rollover
```

ChatGPT 页面应该被视为：

> **Disposable Execution Surface**

真正持久的运行状态必须在 NOOS 中。

---

# 14. 浏览器插件如何参与？

第一版最自然的入口仍然是 Browser Shuttle。

它不应该把业务状态塞在 content script 中，而应分层：

```text
ChatGPT Page
    ↕
Content Script / ChatGPT Adapter
    ↕
Extension Background
    ↕
NOOS Harness Runtime
    ├─ Run Store
    ├─ State Reducer
    ├─ Context Compiler
    ├─ Action Policy
    └─ Performance Controller
```

ChatGPT Adapter 只负责页面能力：

- 识别 conversation；
- 判断是否正在生成；
- 读取最后一条 assistant message；
- 输入 prompt；
- send / stop；
- 页面重连；
- conversation URL / ID。

这样未来 ChatGPT 页面改版，主要坏的是 Adapter，而不是整个 Harness。

未来也可以拥有：

```text
Adapters
├─ ChatGPT
├─ Claude
├─ Gemini
├─ Claude Code
└─ Codex
```

---

# 15. v0 可以利用 Harness Control Block

第一版未必需要额外调用一个 controller model。

Worker ChatGPT 可以在正常回答末尾，同时提出一个机器可读的 control proposal：

```text
<!-- NOOS:CONTROL:BEGIN -->
{
  "base_state_version": 17,
  "progress": "advanced",
  "next_action": "continue_focused",
  "checkpoint_recommended": false,
  "human_gate": null,
  "state_delta": []
}
<!-- NOOS:CONTROL:END -->
```

Shuttle 捕获以后：

```text
parse
→ validate
→ reducer
→ policy engine
→ execute
```

关键点是：

> ChatGPT 给的是 proposal，不是命令。

最终执行权仍然属于 NOOS。

后续如果需要更高可靠性，再加入独立的 Shadow Controller，对 Worker 的完成判断、State Delta 和下一步动作做二次审查。

---

# 16. Performance Controller 不应该交给 LLM

网页是否变卡，是浏览器可以直接观测的问题，不需要让模型猜。

可以综合：

- message 数量；
- DOM node 数；
- Long Task 频率；
- 输入/滚动延迟；
- 可获得时的内存趋势；
- refresh 后是否恢复；
- 插件 observer 自身开销。

形成简单健康状态：

```text
GOOD
DEGRADED
CRITICAL
```

策略例如：

```text
DEGRADED
→ 等待 Safe Window
→ Refresh

Repeatedly DEGRADED after Refresh
→ 增加 Rollover Pressure
```

Round Count 只能是 signal，不能成为“每 20 轮强制换房间”的硬规则。

---

# 17. 多 Conversation 正交审查

在单线程 Harness 稳定以后，第二阶段才加入 Reviewer。

原因很简单：

如果一个 Chat 自己都不能稳定跑，那么开四个 Chat 只会制造四倍的管理问题。

稳定以后，可以建立：

```text
                 Run State
                    │
             Review Snapshot
                    │
       ┌────────────┼────────────┐
       ↓            ↓            ↓
 Domain Review  Product Review  Runtime Review
       │            │            │
       └────────────┼────────────┘
                    ↓
             Structured Issues
                    ↓
              Owner Adjudicate
                    ↓
             Run State Delta
```

Reviewer 不应该得到整个历史聊天。

它只得到为了自己的 review dimension 专门编译的 Context Projection。

例如：

- Domain Reviewer 重点获得术语、因果模型、domain constraints；
- Product Reviewer 重点获得玩家可观察性、策略空间和体验目标；
- Production Reviewer 重点获得 Current Production Facts 与兼容性约束。

这就是 Context Compiler 在多 Agent 场景中的自然延伸。

---

# 18. Reviewer 结果也要结构化

不要把三个 Reviewer 的长篇自然语言全文直接塞回 Main。

应该提炼成 Review Issue：

```yaml
issue_id: R-037
dimension: domain-semantics
severity: blocker

target:
  decision: D-021

claim:
  current model conflates X and Y

evidence:
  - ...

suggested_action:
  reopen_decision
```

然后由 Owner Thread 决定：

```text
ACCEPT
REJECT
DEFER
NEEDS_HUMAN
```

这样 Review 才是一条可管理的工作流，而不是几个 AI 互相复制文章。

---

# 19. 外部知识源为什么应该晚一点做？

长期来看，NOOS 当然应该能从：

- Notion；
- Google Drive；
- GitHub；
- Local Markdown；
- Wiki；
- Reference Library；

读取工作上下文。

但这不是 Harness v0 的前置条件。

原因是如果第一版就开始做 OAuth、权限、写回、RAG、Provider adapter，产品会迅速失焦。

更合理的顺序是：

1. 先证明一个 Run 能自动持续工作；
2. 再证明 NOOS 能通过 compaction + rollover 改善 context fidelity；
3. 再证明多 Reviewer orchestration 有价值；
4. 最后把外部 Source 接入 Context Compiler。

届时外部资料并不是简单“做 RAG”，而是参与：

> **为当前 Action 编译最合适的 Context Projection。**

---

# 20. MVP：只验证一个 Golden Path

第一版应该极端克制。

用户在当前 ChatGPT 对话里点击：

> **Take Over This Thread**

Harness：

```text
Capture current conversation
↓
Create Run
↓
Create Main Logical Thread
↓
Attach current ChatGPT conversation as Execution Session
↓
Extract Initial State
↓
Start Auto Run
```

运行中：

```text
Assistant Response
↓
Control Proposal
↓
State Delta
↓
Reducer
↓
Action Policy
↓
CONTINUE_FOCUSED
```

页面变慢：

```text
DEGRADED
↓
Safe Refresh
↓
Reattach
↓
Fingerprint last message
↓
Resume
```

Context 开始变重或进入新阶段：

```text
Compaction
↓
Checkpoint
↓
Context Compiler
↓
Create Conversation B
↓
Inject Projection
↓
Resume Same Run
```

最后：

```text
Deliverable satisfied
+ Open Questions sufficiently closed
+ No blocker
↓
COMPLETE
```

用户回来以后看到的不是一堆底层操作，而是：

```text
Run completed

31 autonomous rounds
2 safe refreshes
1 conversation rollover
4 decisions added
6 questions closed
1 item requires your decision
```

如果这一条 Golden Path 能稳定工作，这个产品假设已经很有价值。

---

# 21. v0 暂时明确不做什么

为了防止 NOOS 再次长成“大而全平台”，第一阶段明确不做：

- 不做完整知识库；
- 不做 Notion / Drive 全量同步；
- 不做 MCP Server 作为前置条件；
- 不做所有 Chatbot；
- 不做复杂 Multi-Agent Swarm；
- 不做完整可视化 Workflow Builder；
- 不做自动执行高风险外部写操作；
- 不声称控制 ChatGPT 隐藏 context window。

ChatGPT-first 即可。

---

# 22. MVP Acceptance Criteria

不能用“成功自动发送了 30 次继续”作为验收标准。

真正需要验证的是下面这些。

## 22.1 Continuity

一个 Run 跨两个以上 ChatGPT conversation 后，用户仍认为它是同一项连续工作。

## 22.2 State Fidelity

已经 Confirmed 的 constraints / decisions 不会因为 compaction 或 rollover 静默丢失、变义。

## 22.3 Negative Memory

重要 Rejected 方案不会无理由重新出现。

## 22.4 Progress

自主轮次能够关闭问题、推进 Frontier，而不是单纯增加文字。

## 22.5 Performance

用户不需要因为 ChatGPT 页面逐渐卡顿而手动重建整项讨论。

## 22.6 Recovery

刷新、关闭 tab、浏览器重启后，Run 可以恢复到明确状态。

## 22.7 Human Attention

用户不需要全程守着；主要在真正 Authority Boundary 被叫回来。

---

# 23. 必须做真实 Eval，而不是凭感觉

Harness 最重要的产品假设是：

> **External State + Stateful Compaction + Context Projection + Rollover，是否真的比一个无限增长的长 Chat 更好？**

应该拿真实复杂设计任务做 A/B。

### Baseline

```text
一个 ChatGPT conversation
+ 人工不断继续
```

### Harness

```text
NOOS Run
+ State Extraction
+ Compaction
+ Context Compiler
+ Rollover
```

比较：

| 指标 | Baseline | Harness |
|---|---:|---:|
| Decision retention | | |
| Constraint violation | | |
| Rejected reopen rate | | |
| 重复讨论比例 | | |
| Open Question closure | | |
| Useful progress / round | | |
| 人工介入次数 | | |
| 页面性能 | | |
| Active context size | | |

如果 Harness 没有明显改善这些指标，它只是把“自动点击继续”做复杂了。

如果显著改善，才证明 Harness 的核心成立。

---

# 24. 重新看 NOOS 的整体结构

此前 NOOS 更像一个 Context / Artifact Hub：

- Vault；
- Handoff；
- Crystal；
- Result；
- Artifact；
- Context Pack；
- Browser Shuttle。

这些仍然成立。

现在真正补出来的是第二个平面：**Execution Plane**。

```text
NOOS
│
├─ Context / Knowledge Plane
│  ├─ Vault
│  ├─ Crystal
│  ├─ Handoff
│  ├─ Artifact
│  ├─ Reference
│  └─ External Sources
│
└─ Execution Plane
   └─ NOOS Harness
      ├─ Run State
      ├─ State Reducer
      ├─ Context Compiler
      ├─ Action Policy
      ├─ Session Continuity
      ├─ Review Orchestration
      └─ Tool Adapters
```

这两个平面的关系是：

> Knowledge Plane 回答“我们有什么上下文和产物”；Execution Plane 回答“拿到上下文以后，一项 AI 工作怎样持续运行”。

这比把 NOOS 做成 all-in-one AI 工具更稳定：NOOS 不替代 ChatGPT、Claude、Codex，而是在它们之间持有用户自己的工作连续性。

---

# 25. 命名：怒思、怒撕、努思、灵识，怎么选？

命名不是 Harness 架构的一部分，但它会反过来影响产品第一印象，因此值得单独判断。这里先区分“产品语义是否贴合”和“名称是否可占有”两件事；以下只是产品命名判断与快速公开检索，不等同于正式商标 clearance。

## 25.1 “怒思”

这是当前最值得继续验证的中文主名候选。

它的优势首先在于记忆点。`怒` 本身有明显张力，`思` 又把这种张力从“攻击别人”拉回到“强烈、持续地思考”。它和 NOOS 的发音关系也仍然存在，但不像纯音译那样没有中文语义。

它尤其贴合 Harness 的工作方式：

> 不是让模型回答一次，而是让一项复杂工作持续思考、继续推进、反复审查，直到真正收敛。

相比“努思”，“怒思”更像一个具有性格的产品名；相比“怒撕”，它又少了直接冲突和撕扯的攻击性。

主要风险有两个：

- `怒` 仍然是负面情绪字，第一次看到的人可能先联想到愤怒，而不是高强度思考；
- 在医疗、中医情志等语境里，“怒、思”也可以作为两个情志词并列出现，因此中文搜索里会出现少量无关噪音。

快速公开检索暂未发现明显的同类 AI / 软件产品以“怒思”作为强品牌使用；搜索结果更多来自人名、餐饮或食品主体。但这不能替代正式商标、域名和应用商店检索。

**当前判断：如果希望 NOOS 有较强个性和记忆点，“怒思”是目前最好的中文主名候选。**

## 25.2 “怒撕”

优点：

- 极容易记；
- 与 NOOS 有发音联想；
- 很有开发者内部文化；
- “把复杂问题撕开来分析”甚至有一点产品隐喻。

问题：

- 攻击性明显；
- 玩梗感强；
- 很容易先被理解为情绪冲突或“撕某人”；
- 当产品进入企业知识、Agent orchestration、长期工作上下文后，气质会显得过窄。

**结论：非常适合内部昵称 / codename / 彩蛋，不建议做正式中文主品牌。**

## 25.3 “努思”

“努思”最大的优点是安全、直白：努力思考、持续思考，与 Harness 的语义天然兼容。

但重新检索后，它的问题比之前判断的更明显：中文市场已经存在长期使用“努思”的企业管理咨询 / 猎头品牌，公开资料还显示其曾申请并注册“努思 / Nush”相关商标；此外也存在新的“努思培训”等主体。即使这些业务与 AI Harness 并不完全同类，也会降低品牌独占感，并增加后续商标 clearance 的复杂度。

另外，从纯产品感受上说，“努思”也更像教育、培训或咨询品牌，个性弱于“怒思”。

**结论：语义合适，但当前不再作为首选。**

## 25.4 “灵识”

优点：

- 语义漂亮；
- 有“感知、智慧、认知”的感觉；
- 和 AI / context / memory 很自然。

问题：

- 偏玄学与精神性；
- 本身就是已有汉语词；
- AI / 科技领域已经有多个产品、平台和概念使用；
- 搜索辨识度和品牌可占有性较弱。

**结论：语义很好，但不建议作为 NOOS 主品牌。**

## 25.5 “努斯”

如果 NOOS 本身有意呼应哲学术语 `nous`，那么“努斯”作为音译来源非常正。

但它的问题是中文用户第一次看到几乎无法从字面理解产品，而且整体偏学术、古典，更适合作为词源解释，而不是产品名。

## 25.6 当前排序

综合产品语义、记忆度、个性和快速公开检索后的可占有性，当前建议是：

1. **怒思**：正式中文名当前首选候选；
2. **努思**：语义稳妥，但已有品牌使用，降级；
3. **灵识**：语义漂亮，但过于通用且已有大量占用；
4. **怒撕**：保留内部昵称 / codename。

因此现阶段可以使用：

> **NOOS / 怒思**

并把“怒撕”保留成社区和开发阶段的内部玩笑。

英文 `NOOS` 本身也仍然需要正式 clearance：截至 2026 年，已经存在同名 `noos` 的 LLM-agent reliability 项目，而且功能方向涉及 scope drift、circuit break、跨 session state 等，与当前 NOOS Harness 处于明显相邻领域。因此真正公开发布之前，英文名和中文名都应统一做商标、域名、GitHub、PyPI、npm、crates、应用商店和同类产品检索。

---

# 26. 下一步应该设计什么？

到这个阶段，不建议继续横向增加 feature。

下一步应该把三个底层 contract 正式化：

1. **Harness Control Contract v0**  
   Chatbot 每轮如何给出 progress、State Delta 和 next-action proposal。

2. **State Delta Schema + Reducer Rules v0**  
   哪些字段可变、哪些不可静默覆盖、Decision/Rejection/Open Question 如何有生命周期。

3. **Continuation / Recovery State Machine v0**  
   如何从 response complete 走到 continue / refresh / compact / rollover / human gate / complete，并保证幂等与崩溃恢复。

完成这三项后，就已经足够进入一个可实现的浏览器插件 + 本地 runtime MVP。

---

# 27. 最终判断

NOOS Harness 的价值并不在“替用户多点几次继续”。

真正的价值是把 Chatbot 从一个需要人长期照看的聊天窗口，变成一个可以被外部系统监管、恢复、压缩、换代和编排的推理执行器。

它解决的是一类正在越来越明显的问题：

> **模型已经足够聪明，但复杂 AI 工作还缺一个可靠的运行时。**

NOOS 可以不拥有模型，却拥有 Run；不控制 ChatGPT 的隐藏上下文，却拥有 authoritative working state；不保存所有历史作为 active memory，却通过 Context Compiler 决定下一步真正需要什么。

如果这条路线成立，NOOS 的产品核心就会从“跨工具搬运上下文”进一步变成：

> **让用户的 AI 工作在不同模型、不同 conversation、不同 session 之间仍然连续。**

这就是 NOOS Harness 当前最值得验证的核心假设。
