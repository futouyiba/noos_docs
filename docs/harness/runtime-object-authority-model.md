# NOOS Harness｜Runtime Object Model & Authority Model v0

> **状态**：Design Candidate v0  
> **日期**：2026-08-27  
> **上位文档**：[NOOS Harness Overview v0.2](overview.md)  
> **下一层接口**：[State Delta + Reducer Contract v0](state-delta-reducer-contract.md)

---

## 0. 这篇文档解决什么问题

Harness Overview 已经确定一个核心方向：

> **Run 是用户真正拥有的长期工作；Provider Conversation 只是其中一段可替换的执行载体。**

如果不继续把对象边界和 authority 讲清楚，后续所有实现都会混乱：

- 浏览器刷新时，什么应该活下来？
- Conversation rollover 后，旧 conversation 是“丢掉”还是“保留为 provenance”？
- 模型说“这个结论已经确认”，它有没有资格确认？
- Notion / GitHub 当前事实与 Run State 冲突时听谁的？
- Reviewer 基于 v37 提出 blocker，但主线程已推进到 v44，如何判断 issue 是否过期？
- 系统崩溃以后，如何知道上一条 Continue 是否已经发送？

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

## 1.3 模型不能直接拥有 Committed State Transition

LLM 可以发现、推导、建议和提交 Proposal，但不能因为自己写出一句结论，就自动把它升级为正式状态。

> **LLM proposes; Policy authorizes; Reducer applies; NOOS records.**

## 1.4 原始轨迹保留，但不默认进入下一轮上下文

Transcript、旧 conversation、review trace 都应可回溯，但：

> **Evidence Archive ≠ Active Working Context。**

## 1.5 Recovery 必须建立在幂等性上

系统必须区分：

```text
计划了什么
实际发送了吗
Provider 是否观察到了
结果是否被看见
状态变更是否被授权与应用
```

因此 Runtime 必须有 Execution Journal。

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
   ├─ Source Ref[]
   └─ Execution Journal[]

Browser Session[]
└─ Adapter Attachment
   └─ attaches to one Provider Conversation at a time
```

Browser Session 不应被建模为 Provider Conversation 的永久子对象。它只是当前某个浏览器运行表面正在挂接某个 conversation。

---

# 3. Project

Project 是较长期的工作域或知识域，例如：

```text
NOOS
中鱼升级
SLG 求职
```

Project 可以拥有：

- 多个 Run；
- Project-level source / instruction；
- Vault / Artifact；
- Crystal / Handoff；
- 默认 Authority Policy。

Project 不保存瞬时执行状态，例如“正在等待 ChatGPT 第 14 轮输出”。这属于 Run / Execution Journal。

---

# 4. Run

Run 是 Harness 中最重要的一等对象：

> **一个用户希望持续推进、最终达到某个 deliverable 的 AI 工作实例。**

典型生命周期：

```text
created → active → paused → active → completed
```

也可进入 `cancelled` 或从 `completed` 显式 `reopened`。

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

浏览器刷新、tab 关闭、Browser Session 失效、Provider Conversation rollover、Provider 切换、进程重启，都不能杀死 Run。

---

# 5. Logical Thread

Logical Thread 表示 Run 内持续存在的一条工作角色/正交思考线，例如：

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

## 5.1 v0 单 active carrier invariant

> **v0 中，一个 Logical Thread 同一时刻至多有一个 `current_provider_conversation_id`。**

如果需要 reviewer fan-out，应创建多个 Logical Thread，而不是让一个 Thread 同时拥有多个 current carrier。

该 invariant 让 rollover、resume、checkpoint 和 action routing 保持单义；未来若真实用例证明需要同一 Thread 多 active carrier，再显式升级模型。

---

# 6. Provider Conversation

Provider Conversation 是外部 AI Provider 持久化的一段 conversation carrier，例如 ChatGPT conversation ID 或 Claude conversation ID。

它通常：

- 有稳定 URL / ref；
- Provider 服务端会保存；
- 包含 Turn 历史；
- 是 provenance；
- rollover 后仍需回看；
- 可能被重新 attach。

因此正确描述是：

> **replaceable carrier, externally persisted, provenance-bearing。**

状态可以先保持简单：

```text
active
rolled_over
archived
unavailable
```

`rolled_over` 不表示删除，只表示不再承载 Logical Thread 的当前工作面。

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

---

# 7. Browser Session / Adapter Attachment

Browser Session 才是真正可丢弃的执行环境，可以对应 tab、window、WebView、content-script lifecycle 等。

它可能因 refresh、crash、tab close、browser restart、navigation、extension reload 而结束。

最小语义：

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

Adapter Attachment 表示某个 Browser Session 当前成功识别并挂接到一个 Provider Conversation。它是 runtime state，不是业务 state。

---

# 8. Turn

Turn 是 Provider Conversation 内的消息级交互，至少区分：

```text
user turn
assistant turn
provider-visible adapter control turn（若存在）
```

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

它不是原始聊天，也不是外部事实数据库。

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
    source_refs: []

  meta:
    state_version:
    last_checkpoint_id:
```

## 9.1 为什么叫 Committed State，而不是 Canonical State

`canonical` 保留给 **Canonical Source** 这一 epistemic 概念。

Run 内正式提交的约束、决策和 rejection 称为 **Committed State**，避免出现“canonical state 与 canonical source 冲突”这种语义重载。

Committed State：

- 改动少；
- 需要更高 authority；
- 必须 provenance；
- 不允许静默删除；
- 被 supersede/reopen 时仍保留历史。

Working State：

- 允许 agent 高频更新；
- 可替换、衰减或丢弃；
- 不代表正式决策。

---

# 10. Checkpoint

Checkpoint 是某个时刻 Run State 的冻结版本，并附带恢复所需的最小 runtime reference。

用途：

- crash recovery；
- conversation rollover；
- rollback / audit；
- review snapshot generation；
- external write 前的 safety point。

Checkpoint 不是自然语言 Summary。

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

---

# 11. Source Ref：不要把三个维度混成一个 authority enum

Source Ref 是一等对象，因为事实性 claim 必须能追溯“依据什么、哪个版本、什么时候观察到”。

旧式字段：

```yaml
authority: canonical | current | historical | user_asserted | agent_inferred
```

把“谁产生”“冲突时多权威”“时间状态”混在一起，无法表达一个 source 同时是 `user + canonical + current`。

v0 改为正交维度：

```yaml
source_ref:
  id:
  uri:

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

这些 enum 仍是 Design Candidate，后续可根据真实 provider/source 扩展；但**维度分离本身是 v0 invariant**。

## 11.1 Source Ref 不是 Truth Cache

Source Ref 记录的是一次可追溯来源与观察状态。对于会变化的 source：

- `observed_at` 不能缺；
- `version` / `fingerprint` 在可获得时应记录；
- `freshness` 必须允许被标记为 stale/unknown；
- State 中的事实性 claim 不能因为曾被 summarise 进 State 就永久获得真理权。

---

# 12. Authority Model：至少四种问题不能混在一起

## 12.1 Source / Epistemic Authority

回答：

> 某个 claim 的事实依据应该听谁的？

Authority 解析必须结合 **claim_kind**，不能只靠 source origin 排一个全局序。

### Normative claim

例如用户的：

- goal；
- preference；
- scope；
- 用户明确做出的 decision。

在这些问题上：

```text
explicit user assertion > agent inference
```

通常成立。

### Factual claim

例如：

- 当前 Git 代码用了哪个 index；
- Production runtime 当前行为是什么；
- 某 Notion Current 页面现在写了什么。

这时不能简单使用 `user > source`。应优先依据该事实领域的 canonical/current source；用户输入可以作为 correction proposal 或新的 evidence，但不自动覆盖可验证的当前 source。

因此更准确的原则是：

> **Authority resolver 先判断 claim type，再依据 authority role、temporal status、source identity 与 freshness 决定冲突处理。**

v0 不要求实现通用 resolver，但禁止把所有 claim 塞进一条固定优先级链。

## 12.2 Decision Authority

回答：

> 谁有资格把候选结论提升为 Run 的 Committed Decision？

可能是：

- Human only；
- agent delegated；
- reviewer recommendation + owner approval；
- source-derived deterministic rule。

## 12.3 Action Authority

回答：

> 谁可以执行什么动作？

例如 Continue / Refresh / Rollover 可被 delegated；修改正式文档、删除文件等可能必须 Human Gate。

## 12.4 Scope Authority

回答：

> 谁可以改变 Run 正在解决的问题？

默认应最保守。Agent 可以 `propose_scope_change`，但不能静默改 scope。

---

# 13. Promotion Model

Run 内的内容不应一出生就是 Committed Decision。

推荐最小 promotion path：

```text
observation / inference
        ↓
hypothesis
        ↓
candidate decision
        ↓
policy authorization
        ↓
committed decision
```

一个 Run 可以配置 delegated authority，例如：

```yaml
authority_policy:
  agent_may:
    - add_hypothesis
    - add_evidence
    - set_frontier
    - close_exploratory_question

  human_required:
    - change_scope
    - supersede_committed_decision
    - make_product_preference_choice
    - external_irreversible_write
```

这不是 UI 的“Human Gate 列表”，而是 State Delta 能否被授权的正式语义来源。

---

# 14. State Transition Pipeline

建议统一链路：

```text
Worker / Reviewer / User / Source Observer
              ↓
           Proposal
              ↓
      Authority / Promotion Policy
              ↓
        Authorized Delta
              ↓
            Reducer
              ↓
          Run State vN+1
              ↓
        Audit / Journal Ref
```

Reducer 负责完整性，不负责自行判断谁有资格确认产品决策。

后续 operation、precondition、conflict 与 provenance 的正式契约见：

> [State Delta + Reducer Contract v0](state-delta-reducer-contract.md)

---

# 15. Execution Journal

State Store 与 Execution Journal 的职责不同：

```text
State Store       = 现在是什么
Execution Journal = 刚才发生了什么
```

本文只冻结 Journal 必须存在，以及 action 必须可用稳定 `action_id` 追踪。

不要在这里过早把所有生命周期压成一条 `planned → sent → observed → committed` 单轴状态机，因为：

```text
provider side effect happened
assistant result observed
state proposal accepted/rejected
```

是不同维度。

详细 dispatch、reconciliation、state application 与 idempotency 语义留给《Execution Journal & Recovery Contract v0》。

---

# 16. Review Snapshot 与 Staleness

Reviewer 不应该直接读取不断变化的“最新 Main State”后再把结果当作永久有效。

Review Snapshot 至少应包含：

```yaml
review_snapshot_id:
run_id:
base_state_version:
thread_id:
created_at:
projection_fingerprint:
```

Review Issue 至少引用：

```yaml
review_snapshot_id:
base_state_version:
```

Main State 已推进后，系统必须能判断 issue 是仍适用、需 rebase，还是 stale。

---

# 17. Harness 与 Hub 的归属关系

v0 默认：

```text
NOOS Hub
├─ Vault / Artifact Store
├─ Context Broker
├─ Harness Runtime
└─ Tool Router

Browser Shuttle
└─ Provider Adapter
```

> **Harness Runtime 是 Hub 的 Execution subsystem，不是另一个新的本地中枢 daemon。**

除非后续真实隔离/性能要求证明需要拆服务，否则不要制造第二个本地 authority center。

---

# 18. 本文冻结与不冻结什么

## 冻结的语义边界

1. Run / Logical Thread / Provider Conversation / Browser Session 分离；
2. 一个 Logical Thread v0 同时至多一个 current Provider Conversation；
3. Run State 使用 `Committed` 与 `Working` 分层；
4. Canonical Source 与 Committed State 不混词；
5. Source Ref 将 origin / authority role / temporal status / claim kind 正交拆分；
6. Epistemic Authority 与 Operational Authority 分开；
7. Proposal 必须经过 Policy 才能成为 Authorized Delta；
8. Reducer 负责 apply/invariant，不负责 authority；
9. Execution Journal 是 crash recovery 的必要地基；
10. Review 必须带 base state version。

## 仍然是 Candidate 的部分

- Source Ref 各 enum 的最终取值；
- 通用 authority resolver 是否需要实现；
- Checkpoint 采用 snapshot 还是 version pointer；
- Provider Conversation 的 provider-specific adapter schema；
- Execution Journal 的最终 event model。

---

# 19. 下一步

现在不再继续扩大 Object Model。

下一篇进入：

> **《State Delta + Reducer Contract v0》**

重点回答：

- Proposal 和 Authorized Delta 的边界；
- operation taxonomy；
- `base_state_version` / optimistic concurrency；
- precondition；
- Promotion Policy 如何作用到 operation；
- provenance 怎样引用 Turn / Source Ref；
- Reducer invariant；
- conflict / stale / no-op 怎样表达；
- 一次 Delta 是原子的还是部分应用；
- Audit record 如何生成。

这会是从语义模型进入可实现 Runtime Contract 的第一步。
