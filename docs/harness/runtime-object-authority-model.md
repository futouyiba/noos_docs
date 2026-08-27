# NOOS Harness｜Runtime Object Model & Authority Model v0

> **状态**：Design Candidate v0  
> **日期**：2026-08-27  
> **上位文档**：[NOOS Harness Overview v0.2](overview.md)

---

## 0. 这篇文档解决什么问题

Harness Overview 已经确定一个核心方向：

> **Run 是用户真正拥有的长期工作；Chatbot conversation 只是其中一段执行载体。**

但如果不继续把对象边界和 authority 讲清楚，后续所有实现都会混乱：

- 浏览器刷新时，究竟什么应该活下来？
- ChatGPT conversation rollover 后，旧 conversation 是“丢掉”还是“归档”？
- 某个模型说“这个结论已经确认”，它有没有资格这么确认？
- Notion / GitHub 中的 Current Fact 与 Run State 冲突时听谁的？
- Reviewer 返回一个 blocker 时，如果主线程已经从 v37 推进到 v44，该怎么办？
- 系统崩溃以后，如何知道上一条 Continue 是否已经真正发给 Provider？

因此，这篇文档先冻结两个地基：

1. **Runtime Object Model**：系统里到底有哪些一等对象，它们由谁拥有、生命周期多长、如何关联；
2. **Authority Model**：不同类型的信息、决策和动作，谁有资格提出、确认、修改和覆盖。

这篇不是完整技术 schema，也不是 API 设计。它先把语义所有权钉死。

---

# 1. 核心原则

## 1.1 Run 不等于 Conversation

```text
Run = 用户真正拥有的长期工作
Provider Conversation = 外部 AI 平台上的一段对话载体
Browser Session = 某一时刻承载网页与 adapter 的临时执行表面
```

因此：

> **Run durable；Conversation replaceable；Browser Session disposable。**

## 1.2 State 不等于 Truth

Run State 管的是：

- 这项工作现在认为自己在做什么；
- 已经做了什么决定；
- 还有什么没解决；
- 下一步怎么走。

它并不天然拥有所有外部事实的最终真理权。

因此：

> **Operational Authority 与 Epistemic Authority 必须分开。**

## 1.3 模型不能直接拥有 Canonical State Transition

LLM 可以发现、推导和建议。

但：

> **LLM proposes; Policy authorizes; Reducer applies; NOOS records.**

## 1.4 原始轨迹保留，但不默认进入下一轮上下文

Transcript、old conversation、review trace 都应该可回溯。

但：

> **Evidence Archive ≠ Active Working Context。**

## 1.5 Recovery 必须建立在幂等性上

系统必须区分：

```text
我计划做了什么
我真的发出去了吗
Provider 是否观察到了
结果是否已被 State 接纳
```

因此 Runtime 必须有 Execution Journal。

---

# 2. 顶层对象图

第一版对象模型建议如下：

```text
Project
└─ Run
   ├─ Run State
   ├─ Checkpoint[]
   ├─ Logical Thread[]
   │  ├─ Provider Conversation[]
   │  │  └─ Turn[]
   │  └─ Review Snapshot / Review Issue[]
   │
   ├─ Source Ref[]
   └─ Execution Journal[]

Browser Session[]
└─ Adapter Attachment
   └─ attaches to one Provider Conversation at a time
```

注意：Browser Session 不应该被错误地嵌套成 Provider Conversation 的永久子对象。

它只是“当前有一个浏览器实例正在挂接这个 conversation”。

---

# 3. Project

## 3.1 定义

Project 是较长期的工作域或知识域。

例如：

```text
NOOS
中鱼升级
SLG 求职
```

## 3.2 Project 拥有什么

Project 可以拥有：

- 多个 Run；
- Project-level source / instruction；
- Vault / Artifact；
- 长期 Context / Crystal / Handoff；
- 默认 Authority Policy。

## 3.3 Project 不应该承担什么

Project 不应该直接保存某个正在执行步骤的瞬时状态。

例如：

```text
正在等待 ChatGPT 第 14 轮输出
```

属于 Run / Execution Journal，不属于 Project。

---

# 4. Run

## 4.1 定义

Run 是 NOOS Harness 中最重要的一等对象。

它表示：

> **一个用户希望持续推进、最终达到某个 deliverable 的 AI 工作实例。**

例如：

```text
Run:
《World Condition → Fish Response 产品因果模型》
```

## 4.2 Run 的生命周期

典型：

```text
created
→ active
→ paused
→ active
→ completed
```

也可以：

```text
active
→ cancelled
```

或者：

```text
completed
→ reopened
```

## 4.3 Run 必须拥有的最小语义

```yaml
id:
project_id:
title:
goal:
deliverable:
scope:
status:
created_at:
updated_at:
```

## 4.4 Run 是 durable 的

下面这些都不能杀死 Run：

- 浏览器刷新；
- tab 被关闭；
- Browser Session 失效；
- ChatGPT conversation rollover；
- Provider 切换；
- 系统重启。

只有显式完成、取消或用户删除才结束 Run。

---

# 5. Logical Thread

## 5.1 定义

Logical Thread 表示一个在 Run 内持续存在的工作角色或正交思考维度。

例如：

```text
Main Design
Domain Review
Product Review
Production Review
```

它不是一个 conversation。

同一个 Logical Thread 可以跨多个 Provider Conversation。

## 5.2 为什么需要 Logical Thread

没有 Logical Thread 时：

```text
Chat A
Chat B
Chat C
```

只能知道“有三个 Chat”。

有 Logical Thread 后：

```text
Main Design
├─ Chat A
├─ Chat B
└─ Chat C
```

系统知道这三个 conversation 在语义上属于同一条连续工作线。

## 5.3 Logical Thread 的最小字段

```yaml
id:
run_id:
role:
objective:
status:
current_provider_conversation_id:
created_at:
```

---

# 6. Provider Conversation

## 6.1 定义

Provider Conversation 是外部 AI Provider 保存的一段 conversation carrier。

例如：

```text
ChatGPT conversation abc123
Claude conversation xyz789
```

## 6.2 它不是 disposable runtime resource

Provider Conversation 通常：

- 有稳定 URL / ID；
- Provider 服务端会保存；
- 包含历史 Turn；
- 是重要 provenance；
- rollover 后仍然需要回看；
- 可能被重新 attach。

所以正确描述是：

> **replaceable carrier, externally persisted, provenance-bearing。**

## 6.3 Provider Conversation 的状态

可以非常简单：

```text
active
rolled_over
archived
unavailable
```

`rolled_over` 不代表删除，只代表：

> 这个 conversation 不再承载 Logical Thread 的当前工作面。

## 6.4 最小字段

```yaml
id:
logical_thread_id:
provider:
provider_conversation_ref:
url:
status:
started_at:
ended_at:
rollover_reason:
```

---

# 7. Browser Session

## 7.1 定义

Browser Session 是真正的临时执行环境。

它可以对应：

- 一个 tab；
- 一个 window；
- 一个 WebView；
- 一个 extension content-script attachment；
- 一个页面 lifecycle。

## 7.2 Browser Session 是 disposable

它可以随时因为这些原因结束：

- refresh；
- crash；
- tab close；
- browser restart；
- provider navigation；
- extension reload。

结束以后，Run 不应该受影响。

## 7.3 Browser Session 的最小语义

```yaml
id:
provider:
tab_ref:
attached_provider_conversation_id:
health:
status:
started_at:
last_seen_at:
```

## 7.4 Adapter Attachment

Adapter Attachment 表示：

> 某个 Browser Session 当前已经成功识别并挂接到某个 Provider Conversation。

它是 runtime 状态，不是业务状态。

---

# 8. Turn

## 8.1 定义

Turn 是 Provider Conversation 内的一次消息级交互。

至少区分：

```text
user turn
assistant turn
system-visible adapter control turn（若存在）
```

## 8.2 为什么 Turn 需要 provenance

后面的 State Delta 可能引用：

```text
D-017 来自哪一轮？
Q-014 为什么被关闭？
Reviewer 依据了哪段输出？
```

所以最小应该能追到：

```yaml
provider_conversation_id:
provider_message_ref:
role:
observed_at:
fingerprint:
```

Raw content 可以留在 transcript / archive，不要求全部塞进 State Store。

---

# 9. Run State

## 9.1 定义

Run State 表示：

> **NOOS 当前用于继续这项工作的结构化工作状态。**

它不是原始聊天，也不是外部事实数据库。

## 9.2 建议分层

```yaml
run_state:
  operational:
    goal:
    deliverable:
    scope:
    phase:
    status:

  canonical:
    constraints: []
    decisions: []
    rejected: []

  working:
    hypotheses: []
    open_questions: []
    frontier:
    working_set:
    review_issues: []

  sources:
    source_refs: []

  meta:
    state_version:
    last_checkpoint_id:
```

## 9.3 Canonical 与 Working 不同权限

Canonical State：

- 改动少；
- 需要更高 authority；
- 必须 provenance；
- 不允许静默删除。

Working State：

- 允许 agent 高频更新；
- 可以被替换和衰减；
- 不代表正式决策。

---

# 10. Checkpoint

## 10.1 定义

Checkpoint 是某个时刻 Run State 的冻结版本，并附带恢复所需的最小 runtime reference。

## 10.2 Checkpoint 用途

- crash recovery；
- conversation rollover；
- rollback / audit；
- review snapshot generation；
- before external write safety point。

## 10.3 Checkpoint 与 Summary 不同

Checkpoint 不是一篇摘要。

它应该指向：

```yaml
id:
run_id:
state_version:
logical_thread_heads:
provider_conversation_refs:
execution_journal_cursor:
created_at:
reason:
```

State 内容可以 snapshot 或引用版本化 State Store。

---

# 11. Source Ref

## 11.1 为什么 Source Ref 是一等对象

Run State 里出现一句：

> “Production 当前使用 Scheduled DEF。”

如果只保存这句话，过几天它可能已经过期。

因此 State 必须能指向来源。

## 11.2 最小字段

```yaml
id:
uri:
kind:
authority:
version:
observed_at:
freshness:
content_fingerprint:
```

## 11.3 authority 示例

```text
canonical
current
reference
historical
user_asserted
agent_inferred
```

这些值后续可以再正式化。

当前重要的是：

> **State 中的事实性 claim 必须允许追到它依据什么 source authority。**

---

# 12. Authority Model：四种不同问题不能混在一起

“Authority”至少包含四个维度。

## 12.1 Source Authority

回答：

> 某个事实应该听谁的？

例如：

```text
Current Notion > Historical Wiki
Current Git code > 旧 implementation note
用户明确纠正 > agent 猜测
```

这是 Epistemic Authority。

## 12.2 Decision Authority

回答：

> 谁有资格把一个候选结论升级成 Run 的 Confirmed Decision？

可能是：

- Human only；
- agent delegated；
- reviewer recommendation + owner approval；
- source-derived automatic rule。

## 12.3 Action Authority

回答：

> 谁可以执行什么动作？

例如：

- Continue 可以自动；
- Refresh 可以自动；
- Rollover 可以自动；
- 修改 Notion Current 可能必须 Human Gate；
- 删除 GitHub 文件必须 Human Gate。

## 12.4 Scope Authority

回答：

> 谁可以改变这个 Run 正在解决的问题？

默认应该最保守。

Agent 可以发现“可能需要扩大范围”，但只能提出：

```text
propose_scope_change
```

不能自己把 scope 改掉。

---

# 13. Promotion Model

Run 内很多内容都不是一出生就是 Decision。

更合理的生命周期：

```text
observation
↓
hypothesis
↓
candidate conclusion
↓
candidate decision
↓
confirmed decision
```

某些路径也可以：

```text
hypothesis
→ rejected
```

或者：

```text
confirmed decision
→ reopened
→ superseded
```

## 13.1 为什么 Promotion 很重要

如果模型一说：

> “因此我们可以确定……”

系统就自动 `add_decision`，那 Reducer 再严格也没有意义。

所以 State Delta Proposal 必须表达：

```yaml
op: propose_decision
```

而不是所有 agent 都能直接：

```yaml
op: add_confirmed_decision
```

## 13.2 默认 Authority Policy 候选

第一版可以非常保守：

```yaml
agent_may:
  - add_observation
  - add_hypothesis
  - add_evidence
  - open_question
  - set_frontier
  - update_working_set
  - close_exploratory_question

policy_may_auto_promote:
  - low-risk procedural state

human_required:
  - product_choice
  - change_scope
  - supersede_confirmed_decision
  - reopen_rejected_core_option
  - external_write
  - irreversible_action
```

后续可按 Run 类型配置 delegated authority。

---

# 14. State Delta 与 Authority 的关系

推荐链路：

```text
Worker / Reviewer / Tool
      ↓
Proposal
      ↓
Normalize
      ↓
Authority Policy
      ↓
Authorized Operation
      ↓
Reducer
      ↓
State Version N+1
      ↓
Audit / Journal
```

这里每一层回答不同问题：

### Proposal

“模型想改什么？”

### Authority Policy

“它有没有资格这么改？”

### Reducer

“这个改动在状态机上合法吗？”

### Audit / Journal

“这次到底发生了什么？”

不能把这四个职责揉成一个 `apply_delta()`。

---

# 15. Reducer 应维护的最小 invariant

第一版建议冻结：

1. Confirmed Decision 不得静默删除；
2. Confirmed Decision 只能 `supersede` / `reopen`，并保留历史；
3. Constraint 不得由普通 worker 随手改写；
4. Rejected core option 不得无理由复活；
5. Canonical operation 必须有 provenance；
6. base_state_version 必须匹配或进入 reconcile；
7. Working hypothesis 不得被 reducer 自动视为 confirmed；
8. Source-backed factual claim 必须保留 Source Ref；
9. stale review issue 不得直接改写最新 State；
10. external write 不得绕过 Action Authority。

---

# 16. Review Snapshot 与 stale issue

Reviewer 不应该直接消费主线程完整历史。

更合理：

```text
State v37
↓
Review Snapshot RS-008
↓
Reviewer
↓
Review Issue
```

Review Issue 必须带：

```yaml
base_state_version: 37
review_snapshot_id: RS-008
```

如果返回时 Main 已经到 v44：

```text
Issue target 是否仍存在？
相关 Decision 是否已经 superseded？
证据是否过期？
```

Harness 必须先做 stale check，再进入 Owner Adjudication。

---

# 17. Execution Journal

## 17.1 为什么 State Store 不够

State Store 只能告诉我们：

> “现在认为自己处于什么状态。”

但 crash recovery 还需要知道：

> “上一条外部副作用到底执行到哪一步？”

## 17.2 最小 Journal

```yaml
action_id:
run_id:
logical_thread_id:
provider_conversation_id:
browser_session_id:
base_state_version:
action_type:
status: planned | sent | observed | committed | failed
provider_message_ref:
created_at:
updated_at:
```

## 17.3 幂等恢复例子

```text
A-109 planned
↓
prompt 已发给 ChatGPT
↓
A-109 sent
↓
浏览器崩溃
```

重启以后，系统不能直接再次发送 A-109。

它先：

```text
reattach conversation
↓
find provider message / fingerprint
↓
若已存在：observed
若不存在：根据 policy 决定 resend / ask human
```

这就是 Runtime Recovery，而不是普通“刷新后继续”。

---

# 18. Browser Adapter 应该拥有什么，不应该拥有什么

Browser Shuttle / Adapter 负责：

- 识别 Provider Conversation；
- 观察 streaming / completion；
- 获取可见 message；
- 注入 prompt；
- Safe Refresh；
- reattach；
- 页面健康 telemetry。

它不应该拥有：

- Run State 真相；
- Decision Authority；
- Context Compilation Policy；
- Project Vault；
- Review adjudication。

因此浏览器页面永远只是 execution surface。

---

# 19. Hub 应该拥有什么

第一阶段直接定义：

```text
NOOS Hub
├─ Run Store
├─ State Store
├─ Source / Vault
├─ Context Compiler
├─ Authority Policy
├─ Reducer
├─ Action Policy
├─ Execution Journal
└─ Harness Runtime
```

Shuttle 则是：

```text
Browser Shuttle
└─ Provider Adapter
```

Harness Runtime 是 Hub 的 Execution subsystem，不新增第二个本地 daemon。

---

# 20. v0 不冻结的东西

这篇文档故意不冻结：

- 数据库选型；
- JSON / SQLite / event sourcing 的具体实现；
- Provider Adapter API 细节；
- Context Compiler 排序算法；
- Page Health 各指标权重；
- Promotion Policy 的最终 DSL；
- Review Issue 全量 schema；
- 多 Provider 切换策略。

这些都应该在核心对象和 authority 经真实任务验证以后再定。

---

# 21. v0 Acceptance Criteria

这套对象与 authority 模型至少应该让下面这些问题都有唯一答案：

### 页面刷新

Run 是否存在？——存在。  
Conversation 是否存在？——存在。  
Browser Session 是否存在？——旧的结束，新的建立。

### Conversation Rollover

Run 是否变化？——不变。  
Logical Thread 是否变化？——不变。  
Provider Conversation 是否变化？——新增一个，旧的标记 rolled_over。

### Agent 提议重大 Decision

是否直接写入 confirmed？——否。  
先经过什么？——Authority / Promotion Policy。

### 外部 Current Source 更新

旧 State 是否自动变真？——否。  
需要什么？——重新观察 Source，并更新相关 claim / freshness。

### Browser 崩溃

如何防止重复发送？——Execution Journal + provider observation + idempotent recovery。

### Reviewer 返回旧问题

是否直接应用？——否。  
先检查什么？——base_state_version / snapshot freshness / target validity。

如果这些语义仍然存在歧义，这个模型就还没有冻结资格。

---

# 22. 下一步

对象和 authority 作为地基确定后，下一篇应进入：

> **State Delta + Reducer Contract v0**

重点正式化：

- State schema；
- Delta operation vocabulary；
- propose / authorize / apply 边界；
- version conflict；
- provenance；
- reopen / supersede；
- stale review handling；
- reducer invariant。

之后再写：

1. Continuation State Machine v0；
2. Execution Journal & Recovery Contract v0。

---

## 当前最短结论

> **Run 是工作；Logical Thread 是持续角色；Provider Conversation 是可替换载体；Browser Session 是可销毁执行表面。**
>
> **Source 决定事实权威；Run State 决定工作状态；LLM 提议；Policy 授权；Reducer 执行；NOOS 记录。**
