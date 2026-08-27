# NOOS Harness｜Runtime Object Model & Authority Model v0

> **状态**：Design Candidate v0  
> **日期**：2026-08-27  
> **上位文档**：[NOOS Harness Overview v0.2](overview.md)  
> **下一层接口**：[State Delta + Reducer Contract v0](state-delta-reducer-contract.md)

---

## 0. 这篇文档解决什么问题

Harness Overview 已经确定一个核心方向：

> **Run 是用户真正拥有的长期工作；Provider Conversation 只是其中一段可替换的执行载体。**

如果不把对象边界和 authority 钉清楚，后续 Contract 会在几个地方迅速混乱：

- 浏览器刷新时，什么应该活下来？
- Conversation rollover 后，旧 conversation 是“丢掉”还是“保留为 provenance”？
- 模型说“这个结论已经确认”，它有没有资格确认？
- Notion / GitHub 当前事实与 Run State 冲突时听谁的？
- 一个 Source 到底是什么，以及它对某个具体 Claim 为什么有权威性？
- Reviewer 基于 v37 提出 blocker，但主线程已推进到 v44，如何识别 stale issue？
- 系统崩溃以后，如何知道上一条动作是否已经发出？

因此本文冻结两个地基：

1. **Runtime Object Model**：有哪些一等对象、谁拥有、生命周期多长、如何关联；
2. **Authority Model**：事实、决策、动作与 scope 分别由谁产生、确认、修改和覆盖。

本文不是完整 API/schema，而是后续 Contract 的语义地基。

---

# 1. 核心原则

## 1.1 Run 不等于 Conversation

```text
Run = 用户真正拥有的长期工作
Provider Conversation = 外部 AI Provider 上的一段对话载体
Browser Session = 某一时刻承载网页与 adapter 的临时运行表面
```

因此：

> **Run durable；Provider Conversation replaceable；Browser Session disposable。**

## 1.2 State 不等于 Truth

Run State 管：

- 这项工作现在在做什么；
- 已经提交了哪些约束与决策；
- 还有什么未解决；
- 下一步怎么继续。

外部事实是否为真，仍由相应 source 的 epistemic authority 决定。

> **Operational Authority 与 Epistemic Authority 必须分开。**

## 1.3 Source 不等于 Claim

同一篇文档可以同时包含事实、约束、决策与推断；同一个 Git commit 也可以被不同 Claim 以不同方式引用。

因此：

> **SourceRef 表示“我观察了什么”；EvidenceRef 表示“这次观察对某个 Claim 提供了什么证据、具有什么 authority role”。**

`claim_kind` 和 `authority_role` 不再属于 SourceRef 本体。

## 1.4 模型不能直接拥有 Committed State Transition

LLM 可以发现、推导、建议和提交 Proposal，但不能因为自己写出一句结论，就自动把它升级为正式状态。

> **LLM proposes; Policy authorizes; Reducer applies; NOOS records.**

## 1.5 原始轨迹保留，但不默认进入下一轮上下文

Transcript、旧 conversation、review trace 都应可回溯，但：

> **Evidence Archive ≠ Active Working Context。**

## 1.6 Recovery 必须建立在幂等性上

系统至少必须区分：

```text
Provider side effect 是否发生
Assistant result 是否观察到
State Proposal 是否接受并应用
```

这几个维度不能在 Object Model 阶段硬压成一个单轴状态机。具体事件语义由后续 Execution Journal & Recovery Contract 定义。

---

# 2. 顶层对象图

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
   ├─ SourceRef[]
   ├─ EvidenceRef[]
   ├─ AuthorizationResult[]
   │  └─ PendingHumanGate? 
   └─ Execution Journal[]

Browser Session[]
└─ Adapter Attachment
   └─ attaches to one Provider Conversation at a time
```

Browser Session 不嵌套成 Provider Conversation 的永久子对象。它只是“当前有一个运行表面正在挂接这个 conversation”。

---

# 3. Project

Project 是较长期的工作域或知识域，例如：

```text
NOOS
中鱼升级
SLG 求职
```

Project 可以拥有多个 Run、Project-level source/instruction、Vault/Artifact、长期 Crystal/Handoff，以及默认 Authority Policy。

Project 不保存某个具体执行步骤的瞬时状态，例如“正在等待 ChatGPT 第 14 轮输出”；这属于 Run / Execution Journal。

---

# 4. Run

Run 是 Harness 最重要的一等对象：

> **一个用户希望持续推进、最终达到某个 deliverable 的 AI 工作实例。**

最小语义：

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

典型生命周期：

```text
created → active → paused → active → completed
```

也允许 `cancelled` 与显式 `reopened`。

浏览器刷新、tab 关闭、Browser Session 失效、Conversation rollover、Provider 切换、系统重启都不能杀死 Run。

---

# 5. Logical Thread

Logical Thread 表示 Run 内持续存在的工作角色或正交思考维度，例如：

```text
Main Design
Domain Review
Product Review
Production Review
```

同一个 Logical Thread 可以跨多个 Provider Conversation。

最小字段：

```yaml
id:
run_id:
role:
objective:
status:
current_provider_conversation_id:
created_at:
```

## 5.1 v0 invariant：单 active carrier

> **一个 Logical Thread 同一时刻至多有一个 `current_provider_conversation_id`。**

Rollover 是把 current carrier 从 A 切到 B，而不是让 A/B 同时成为 current。

Reviewer fan-out 应创建多个 Logical Thread，而不是把一个 Logical Thread 的 current conversation 改成数组。

---

# 6. Provider Conversation

Provider Conversation 是外部 AI Provider 保存的一段 conversation carrier，例如 ChatGPT conversation 或 Claude conversation。

它通常：

- 有稳定 URL / ID；
- Provider 服务端会持久化；
- 包含历史 Turn；
- 是 provenance；
- rollover 后仍需回看；
- 可能重新 attach。

因此正确描述是：

> **replaceable carrier, externally persisted, provenance-bearing。**

状态可先保持很小：

```text
active
rolled_over
archived
unavailable
```

最小字段：

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

`rolled_over` 不代表删除，只代表不再承载该 Logical Thread 的当前工作面。

---

# 7. Browser Session / Adapter Attachment

Browser Session 是真正的临时执行环境，可以对应 tab、window、WebView、content-script attachment 或一次页面 lifecycle。

它可以因为 refresh、crash、tab close、browser restart、provider navigation、extension reload 随时结束。

最小字段：

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

Adapter Attachment 表示某个 Browser Session 当前已经成功识别并挂接到某个 Provider Conversation。它是 runtime 状态，不是业务状态。

---

# 8. Turn

Turn 是 Provider Conversation 内的一次消息级交互。

最小 provenance：

```yaml
provider_conversation_id:
provider_message_ref:
role:
observed_at:
fingerprint:
```

Raw content 可以留在 transcript/archive，不要求全部复制进 State Store。

---

# 9. Run State

Run State 表示：

> **NOOS 当前用于继续这项工作的结构化工作状态。**

建议分层：

```yaml
run_state:
  operational:
    goal:
    deliverable:
    scope:
    phase:
    status:

  committed:
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
    source_ref_ids: []
    evidence_ref_ids: []

  meta:
    state_version:
    last_checkpoint_id:
```

`Committed State` 是 Run 内正式提交的治理状态；`Canonical Source` 是外部事实的权威来源。两者不再共用 `canonical` 这个词。

Committed State：改动少、需要更高 authority、必须 provenance、不允许静默删除。

Working State：允许更高频更新、可以被替换和衰减、不代表正式决策。

---

# 10. Checkpoint

Checkpoint 是某个时刻 Run State 的冻结版本，并附带恢复所需的最小 runtime reference。

用途包括：crash recovery、conversation rollover、rollback/audit、review snapshot generation、external write 前安全点。

最小字段：

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

Checkpoint 不是摘要；State 内容可以 snapshot 或引用版本化 State Store。

---

# 11. SourceRef：我观察了什么

## 11.1 定义

SourceRef 表示一个**可追溯、具有内容身份的 source observation**。

它回答：

> “这次工作观察了哪个来源、哪个版本、在什么时间看到的内容？”

最小字段：

```yaml
source_ref:
  id:
  uri:
  origin_kind: user | document | runtime | agent | external
  version:
  observed_at:
  freshness:
  content_fingerprint:
  supersedes_source_ref_id:
```

## 11.2 Observation identity 必须 immutable

以下与“当时看到了什么”有关的身份字段一旦被 provenance 引用，就不得原地覆盖：

```text
uri/version observation
observed_at
content_fingerprint
```

重新观测同一逻辑来源时，创建新的 SourceRef：

```text
SourceRef S-004 (commit A)
        ↓ superseded by
SourceRef S-011 (commit B)
```

旧 Decision 仍然引用 S-004，因此历史 provenance 不会被今天的观察篡改。

`freshness` 可以由外部派生/重新计算，但不能通过修改内容身份来假装旧 snapshot 是新 snapshot。

---

# 12. EvidenceRef：这个 Source 对哪个 Claim 起什么作用

Source 与 Claim 必须分开。

EvidenceRef 表示：

> **某个 SourceRef 中的某段证据，被用来支持/反驳某个 Claim，并在这个 Claim 上扮演什么 authority role。**

建议字段：

```yaml
evidence_ref:
  id:
  source_ref_id:
  claim_ref:

  claim_kind:
    # fact | preference | constraint | decision | inference

  stance:
    # supports | contradicts | contextualizes

  authority_role:
    # canonical | supporting | reference

  claim_excerpt_fingerprint:
  created_at:
```

因此：

```text
SourceRef
= 我看了什么

EvidenceRef
= 它对这个 Claim 证明了什么，以及在这个 Claim 上具有什么地位
```

同一个 SourceRef 可以产生多个 EvidenceRef，而且这些 EvidenceRef 的 `claim_kind` / `authority_role` 可以不同。

---

# 13. Authority Model：四类问题不能混在一起

## 13.1 Source / Epistemic Authority

回答：

> 某个 factual claim 应该相信什么证据？

不能有一条全局 `user > document > agent` 排序。必须结合 Claim 类型和 EvidenceRef：

```text
Current Notion Production Fact > Historical Wiki
Current Git code > old implementation note
```

但：

```text
用户说“代码现在用 index A”
```

并不会自动覆盖当前 Git code 证明的 index B。

## 13.2 Decision Authority

回答：

> 谁有资格把候选结论升级成 Committed Decision？

可能是 Human only、agent delegated、review recommendation + owner approval、source-derived rule。

## 13.3 Action Authority

回答：

> 谁可以执行什么动作？

例如 Continue/Refresh/Rollover 可委托自动化；修改正式文档、删除文件等可能要求 Human Gate。

## 13.4 Scope Authority

回答：

> 谁可以改变 Run 正在解决的问题？

Agent 可以 `request` scope change，但默认不能自行修改 committed scope。

---

# 14. Claim Type 决定 Authority Resolver 的规则

至少先区分：

### Normative Claim

例如：

- 用户 goal；
- preference；
- scope；
- product decision；
- 用户明确 constraint。

这类 Claim 中，用户/被授权 owner 的表达通常拥有更高 authority。

### Factual Claim

例如：

- 当前 Git 代码使用什么实现；
- Production 当前部署了什么；
- 某 API 当前定义是什么。

这类 Claim 应根据 canonical/current evidence 判断，而不能套用 `user assertion > source`。

### Inference

Agent 推导出的解释默认保持 inference/hypothesis，除非通过 Promotion Policy 进入 Committed State。

---

# 15. Promotion / Authorization Model

Run 内的状态晋升不由 Reducer 自己判断。

典型链路：

```text
Proposal
   ↓
Authority / Promotion Policy
   ↓
AuthorizationResult
   ├─ authorized
   ├─ requires_human
   └─ denied
```

若 authorized，Policy 才产生 Authorized Delta 交给 Reducer。

若 requires_human，Policy 创建 durable PendingHumanGate；Reducer 不执行。

原则：

> **LLM proposes; Policy authorizes; Reducer applies; NOOS records.**

默认 delegated authority 可以允许 agent 更新 Working State；改变 goal/scope、supersede committed decision、不可逆 external write 等通常要求 Human Gate。

---

# 16. AuthorizationResult 与 PendingHumanGate

Human Gate 属于 Policy/Authority 边界，不属于 Reducer 的副作用。

建议最小对象：

```yaml
authorization_result:
  id:
  proposal_id:
  run_id:
  status: authorized | requires_human | denied
  authority_basis:
  gate_id:
  reason:
  created_at:
```

若 `requires_human`：

```yaml
pending_human_gate:
  id:
  run_id:
  proposal_id:
  authorization_result_id:
  gate_kind:
  prompt:
  status: pending | approved | rejected | cancelled
  created_at:
  resolved_at:
```

这些对象必须 durable。否则 Policy 已经判断“需要人”，但 Browser/Hub 在用户回答前崩溃，就会丢失等待中的 authority boundary。

后续 Continuation Runtime 消费的是：

```text
Run State
+ AuthorizationResult / PendingHumanGate
+ ApplyResult（如果 Reducer 实际执行过）
+ Runtime Signals
```

---

# 17. Review Snapshot / Review Issue

Review Snapshot 必须绑定：

```yaml
review_snapshot_id:
base_state_version:
```

Review Issue 至少记录：

```yaml
issue_id:
review_snapshot_id:
base_state_version:
dimension:
severity:
target:
claim:
evidence_ref_ids:
suggested_action:
```

Reviewer 返回时，若主线程已推进，Policy/Review Orchestrator 必须识别 stale；Reducer 不猜测如何 merge。

---

# 18. Execution Journal 的边界

Object Model 只冻结两个事实：

```text
State Store = 现在是什么
Execution Journal = 执行过程中发生了什么
```

并且 Execution Journal 未来不能把不同维度粗暴压成一个 `committed`：

- dispatch/provider side effect；
- observation/result capture；
- reconciliation；
- state proposal/application。

具体 event/status 结构留给 `Execution Journal & Recovery Contract v0`。

---

# 19. Harness 与 Hub 的关系

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

除非以后性能/隔离要求证明必须拆服务，否则 v0 不制造新的本地 daemon。

---

# 20. 当前 v0 Invariants

1. Run 是 durable、NOOS-owned。
2. Logical Thread 是 durable，并且同一时刻至多一个 current Provider Conversation。
3. Provider Conversation replaceable、provider-persisted、provenance-bearing。
4. Browser Session / Adapter Attachment disposable。
5. Committed State 与 Canonical Source 是两个不同概念。
6. SourceRef 与 EvidenceRef 分离：Source ≠ Claim。
7. Source observation 内容身份 immutable；重新观察创建新 SourceRef。
8. Normative Claim 与 Factual Claim 使用不同 Authority 规则。
9. LLM 不能直接写 Committed State。
10. `requires_human` 必须形成 durable AuthorizationResult / PendingHumanGate。
11. Reducer 不负责 Authority、事实判断、Human Gate 或 external side effect。
12. Stale Review 必须显式识别，不能静默 merge。

---

# 21. 与 State Delta Contract 的接口

下一层 Contract 使用这里的对象与语义：

```text
SourceRef
EvidenceRef
Committed State
Working State
Proposal
AuthorizationResult
PendingHumanGate
Authorized Delta
```

特别是：

```text
Proposal = 请求一个最终 State Transition
Operation = 期望发生的 mutation
Policy = 决定这个事务是否有权发生
Reducer = 在授权后确定性地应用 mutation
```

因此 State Delta v0 不再需要 `propose_decision` 之类二次“提议操作”。

---

## 当前状态

本文继续保持 **Design Candidate v0**。

Source/Evidence、Authority、Human Gate ownership 已经收敛；待与 State Delta Contract 完成一次接口对审后，再考虑升为 Design Baseline。
