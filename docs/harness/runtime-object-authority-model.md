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
- Policy 已经产生 Human Gate 后，如果 Hub 崩溃，原 Proposal 还能不能恢复？
- Reviewer 基于 v37 提出 blocker，但主线程已推进到 v44，如何识别 stale issue？

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

Run State 管工作状态；外部事实是否为真，仍由相应 source/evidence 的 epistemic authority 决定。

> **Operational Authority 与 Epistemic Authority 必须分开。**

## 1.3 Source 不等于 Claim

同一篇文档可以同时包含事实、约束、决策与推断；同一个 Git commit 也可以被不同 Claim 以不同方式引用。

> **SourceRef 表示“我观察了什么”；EvidenceRef 表示“这次观察对某个 Claim 提供了什么证据、具有什么 authority role”。**

`claim_kind` 和 `authority_role` 不属于 SourceRef 本体。

## 1.4 Proposal 必须 durable

Proposal 不是一次临时函数调用参数。

如果 Policy 返回 `requires_human`，系统可能在用户批准前经历：

```text
Browser refresh
Hub restart
Provider reattach
```

因此原始 Proposal 必须可恢复：

> **Proposal 是 durable state-transition request；AuthorizationResult / PendingHumanGate 都通过 proposal_id 引用它。**

## 1.5 模型不能直接拥有 Committed State Transition

> **LLM proposes; Policy authorizes; Reducer applies; NOOS records.**

## 1.6 原始轨迹保留，但不默认进入下一轮上下文

> **Evidence Archive ≠ Active Working Context。**

## 1.7 Recovery 必须建立在幂等性上

系统至少区分：

```text
Provider side effect 是否发生
Assistant result 是否观察到
State Proposal 是否接受并应用
```

具体事件语义由后续 Execution Journal & Recovery Contract 定义。

---

# 2. 顶层对象图

```text
NOOS Hub
│
├─ Evidence / Source Store
│  ├─ SourceRef[]
│  └─ EvidenceRef[]
│
└─ Project
   └─ Run
      ├─ Run State
      │  └─ references SourceRef / EvidenceRef IDs
      ├─ Checkpoint[]
      ├─ Logical Thread[]
      │  ├─ Provider Conversation[]
      │  │  └─ Turn[]
      │  └─ Review Snapshot / Review Issue[]
      │
      ├─ Proposal[]
      ├─ AuthorizationResult[]
      ├─ PendingHumanGate[]
      ├─ State Transition / Audit Records[]
      └─ Execution Journal[]

Browser Session[]
└─ Adapter Attachment
   └─ attaches to one Provider Conversation at a time
```

两个 ownership 原则：

1. Browser Session 不嵌套成 Provider Conversation 的永久子对象。
2. SourceRef/EvidenceRef 是 Hub 的可追溯证据对象；Run State 只引用它们，不让 Reducer 原地重写 source observation。

---

# 3. Project

Project 是较长期的工作域或知识域，例如：

```text
NOOS
中鱼升级
SLG 求职
```

Project 可以拥有多个 Run、Project-level source/instruction、Vault/Artifact、长期 Crystal/Handoff，以及默认 Authority Policy。

Project 不保存某个具体执行步骤的瞬时状态。

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

Rollover 把 current carrier 从 A 切到 B；Reviewer fan-out 创建多个 Logical Thread。

---

# 6. Provider Conversation

Provider Conversation 是外部 AI Provider 保存的一段 conversation carrier。

正确描述：

> **replaceable carrier, externally persisted, provenance-bearing。**

状态可保持：

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

Adapter Attachment 表示某个 Browser Session 当前已成功识别并挂接到某个 Provider Conversation。它是 runtime 状态，不是业务状态。

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

`Committed State` 是 Run 内正式提交的治理状态；`Canonical Source` 是外部事实的权威来源。

Committed State 改动少、需要更高 authority、必须 provenance、不允许静默删除。

Working State 允许更高频更新、可以被替换和衰减、不代表正式决策。

---

# 10. Checkpoint

Checkpoint 是某个时刻 Run State 的冻结版本，并附带恢复所需的最小 runtime reference。

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

# 11. Proposal：durable transition request

Proposal 表示：

> **某个 proposer 请求 Run State 发生的一笔最小逻辑事务。**

Proposal 必须 durable，因为：

- AuthorizationResult 引用它；
- PendingHumanGate 引用它；
- Human approval 后 Policy 需要重新读取它；
- crash/restart 后必须能恢复这笔等待中的事务。

最小 identity 由 State Delta Contract 正式定义，至少包括：

```yaml
id:
run_id:
base_state_version:
proposer:
operations: []
created_at:
```

Proposal 不等于已授权 Delta。

---

# 12. SourceRef：我观察了什么

SourceRef 是 Hub Evidence / Source Store 中的**可追溯 source observation**。

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

## 12.1 Observation identity immutable

一旦被 provenance 引用，下列内容身份字段不得原地覆盖：

```text
uri/version observation
observed_at
content_fingerprint
supersedes_source_ref_id
```

重新观察同一逻辑来源时，创建新 SourceRef：

```text
S-004 (commit A)
   ↓ superseded by
S-011 (commit B)
```

`supersedes_source_ref_id` 应在新 observation 创建时确定，而不是事后修改旧 observation。

`freshness` 可以是外部派生状态，但不能通过修改内容身份来假装旧 snapshot 是新 snapshot。

---

# 13. EvidenceRef：这个 Source 对哪个 Claim 起什么作用

EvidenceRef 表示：

> **某个 SourceRef 中的证据，被用于支持/反驳某个 Claim，并在这个 Claim 上扮演什么 authority role。**

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

```text
SourceRef = 我看了什么
EvidenceRef = 它对这个 Claim 证明了什么
```

同一个 SourceRef 可以产生多个 EvidenceRef。

---

# 14. Authority Model：四类问题不能混在一起

## 14.1 Source / Epistemic Authority

回答 factual claim 应相信什么证据。

不能有一条全局 `user > document > agent` 排序，必须结合 Claim 类型和 EvidenceRef。

## 14.2 Decision Authority

回答谁有资格把候选结论升级成 Committed Decision。

## 14.3 Action Authority

回答谁可以执行 Continue / Refresh / Rollover / external write 等动作。

## 14.4 Scope Authority

回答谁可以改变 Run 正在解决的问题。

Agent 可以请求 scope change，但默认不能自行修改 committed scope。

---

# 15. Claim Type 决定 Authority Resolver 的规则

### Normative Claim

用户 goal、preference、scope、product decision、用户明确 constraint。

这类 Claim 中，用户/被授权 owner 的表达通常拥有更高 authority。

### Factual Claim

当前 Git 实现、Production 当前部署、API 当前定义等。

这类 Claim 应根据 canonical/current evidence 判断，不能套用 `user assertion > source`。

### Inference

Agent 推导默认保持 inference/hypothesis，除非通过 Promotion Policy 进入 Committed State。

---

# 16. Promotion / Authorization Model

典型链路：

```text
Durable Proposal
   ↓
Authority / Promotion Policy
   ↓
Durable AuthorizationResult
   ├─ authorized
   ├─ requires_human
   └─ denied
```

若 authorized，Policy 产生 Authorized Delta 交给 Reducer。

若 requires_human，Policy 创建 durable PendingHumanGate；Reducer 不执行。

> **LLM proposes; Policy authorizes; Reducer applies; NOOS records.**

---

# 17. AuthorizationResult 与 PendingHumanGate

Human Gate 属于 Policy/Authority 边界，不属于 Reducer 副作用。

统一 identity 字段：

```yaml
authorization_result:
  id:
  proposal_id:
  run_id:
  base_state_version:
  status: authorized | requires_human | denied
  authority_basis:
  gate_id:
  reason:
  created_at:
  supersedes_authorization_result_id:
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

这些对象必须 durable。

Human approval 后，Policy 应基于原 durable Proposal 重新评估并创建新的 AuthorizationResult，而不是把旧结果原地从 `requires_human` 改为 `authorized`。

---

# 18. Review Snapshot / Review Issue

Review Snapshot 绑定：

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

Reviewer 返回时若主线程已推进，Policy/Review Orchestrator 必须识别 stale；Reducer 不猜测 merge。

---

# 19. State Transition / Audit Records

State Delta Contract 会产生：

```text
AuthorizationResult
AuthorizedDelta
ApplyResult
Audit Record
```

其中 Proposal、AuthorizationResult、PendingHumanGate 必须 durable；AuthorizedDelta / ApplyResult / Audit Record 也必须至少保存在可追溯的 transition/audit store 中，以支持：

- 为什么 State 从 v17 到 v18；
- 哪笔 Authorization 导致这个 transition；
- crash 后是否已经 apply；
- no-op / stale / invariant failure 的审计。

具体 schema 由 State Delta Contract 定义。

---

# 20. Execution Journal 的边界

```text
State Store = 现在是什么
Execution Journal = 执行过程中发生了什么
```

Execution Journal 不能把不同维度粗暴压成一个 `committed`：

- dispatch/provider side effect；
- observation/result capture；
- reconciliation；
- state proposal/application。

具体 event/status 结构留给 `Execution Journal & Recovery Contract v0`。

---

# 21. Harness 与 Hub 的关系

```text
NOOS Hub
├─ Vault / Artifact Store
├─ Evidence / Source Store
├─ Context Broker
├─ Harness Runtime
└─ Tool Router

Browser Shuttle
└─ Provider / Chatbot Adapter
```

> **Harness Runtime 是 Hub 的 Execution subsystem；Shuttle 是浏览器侧执行与连接层。**

v0 不制造新的本地 daemon。

---

# 22. 当前 v0 Invariants

1. Run durable、NOOS-owned。
2. Logical Thread durable，并且同一时刻至多一个 current Provider Conversation。
3. Provider Conversation replaceable、provider-persisted、provenance-bearing。
4. Browser Session / Adapter Attachment disposable。
5. Committed State 与 Canonical Source 是不同概念。
6. SourceRef 与 EvidenceRef 分离：Source ≠ Claim。
7. Source observation 内容身份 immutable；重新观察创建新 SourceRef。
8. Run State 只 attach SourceRef/EvidenceRef IDs，不由 Reducer改写 observation identity。
9. Normative Claim 与 Factual Claim 使用不同 Authority 规则。
10. Proposal 必须 durable。
11. LLM 不能直接写 Committed State。
12. `requires_human` 必须形成 durable AuthorizationResult / PendingHumanGate。
13. Reducer 不负责 Authority、事实判断、Human Gate 或 external side effect。
14. Stale Review 必须显式识别，不能静默 merge。

---

# 23. 与 State Delta Contract 的接口

下一层 Contract 使用：

```text
SourceRef
EvidenceRef
Committed State
Working State
Durable Proposal
AuthorizationResult
PendingHumanGate
Authorized Delta
ApplyResult
```

特别是：

```text
Proposal = 请求一个最终 State Transaction
Operation = 期望发生的 mutation
Policy = 决定这个事务是否有权发生
Reducer = 在授权后确定性地应用 mutation
```

因此 State Delta v0 不需要 `propose_decision` 等二次“提议操作”。

---

## 当前状态

本文继续保持 **Design Candidate v0**。

对象 ownership、Source/Evidence、Authority、durable Proposal 与 Human Gate ownership 已收敛；待与 State Delta Contract 完成接口对审后，再考虑升为 Design Baseline。
