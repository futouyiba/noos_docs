# NOOS Harness｜Runtime Object Model & Authority Model v0

> **状态**：Design Baseline v0.1  
> **日期**：2026-08-27  
> **上位文档**：[NOOS Harness Overview v0.2](overview.md)  
> **下一层接口**：[State Delta + Reducer Contract v0](state-delta-reducer-contract.md)

---

## 0. 这篇文档解决什么问题

Harness 的核心前提是：

> **Run 是用户真正拥有的长期工作；Provider Conversation 只是其中一段可替换的执行载体。**

本文冻结两个地基：

1. **Runtime Object Model**：有哪些一等对象、谁拥有、生命周期多长、如何关联；
2. **Authority Model**：事实、决策、动作与 scope 分别由谁产生、确认、修改和覆盖。

本文不是完整 API/schema，而是后续 Contract 的语义基线。

---

# 1. 核心原则

## 1.1 Run 不等于 Conversation

```text
Run = 用户真正拥有的长期工作
Provider Conversation = 外部 AI Provider 上的一段对话载体
Browser Session = 某一时刻承载网页与 adapter 的临时运行表面
```

> **Run durable；Provider Conversation replaceable；Browser Session disposable。**

## 1.2 State 不等于 Truth

Run State 管工作状态；外部事实是否为真，由相应 source/evidence 的 epistemic authority 决定。

> **Operational Authority 与 Epistemic Authority 必须分开。**

## 1.3 Source 不等于 Claim

> **SourceRef 表示“我观察了什么”；EvidenceRef 表示“这次观察中的某段证据具有什么 claim kind / authority role”。**

`claim_kind` / `authority_role` 不属于 SourceRef 本体。

## 1.4 Proposal 必须 durable 且 immutable

Proposal 是状态事务请求，不是临时函数参数。

如果 Policy 返回 `requires_human`，系统可能在审批前经历 Browser refresh、Hub restart、Provider reattach。因此原 Proposal 必须可恢复，而且审批必须针对**同一笔未被改写的事务**。

> **Proposal 创建后内容不可原地修改；若 operation 内容改变，必须创建新的 Proposal ID。**

## 1.5 模型不能直接拥有 Committed State Transition

> **LLM proposes; Policy authorizes; Reducer applies; NOOS records.**

## 1.6 原始轨迹保留，但不默认进入下一轮上下文

> **Evidence Archive ≠ Active Working Context。**

## 1.7 Recovery 必须建立在幂等性上

系统至少区分 Provider side effect、Assistant result observation、State Proposal/application。具体事件语义由 Execution Journal & Recovery Contract 定义。

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

Project 是较长期工作域或知识域，可拥有多个 Run、Project-level instruction、Vault/Artifact、长期 Crystal/Handoff，以及默认 Authority Policy。

Project 不保存具体执行步骤的瞬时状态。

---

# 4. Run

Run 是 Harness 最重要的一等对象：

> **一个用户希望持续推进、最终达到某个 deliverable 的 AI 工作实例。**

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

浏览器刷新、tab 关闭、Browser Session 失效、Conversation rollover、Provider 切换、系统重启都不能杀死 Run。

---

# 5. Logical Thread

Logical Thread 表示 Run 内持续存在的工作角色或正交思考维度，例如 Main Design、Domain Review、Product Review。

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

Provider Conversation 是外部 AI Provider 保存的一段 conversation carrier：

> **replaceable carrier, externally persisted, provenance-bearing。**

```text
active
rolled_over
archived
unavailable
```

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

`rolled_over` 不代表删除。

---

# 7. Browser Session / Adapter Attachment

Browser Session 是临时执行环境，可以对应 tab、window、WebView、content-script attachment 或一次页面 lifecycle。

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

Adapter Attachment 是 runtime 状态，不是业务状态。

---

# 8. Turn

Turn 是 Provider Conversation 内的一次消息级交互。

```yaml
provider_conversation_id:
provider_message_ref:
role:
observed_at:
fingerprint:
```

Raw content 可以留在 transcript/archive。

---

# 9. Run State

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

`Committed State` 是 Run 内正式治理状态；`Canonical Source` 是外部事实权威来源。

---

# 10. Checkpoint

Checkpoint 是某个时刻 Run State 的冻结版本，并附带恢复所需最小 runtime reference。

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

Checkpoint 不是摘要。

---

# 11. Proposal：durable immutable transition request

Proposal 表示：

> **某个 proposer 请求 Run State 发生的一笔最小逻辑事务。**

```yaml
proposal:
  id:
  run_id:
  base_state_version:
  proposer:
  operations: []
  operations_fingerprint:
  created_at:
```

关键 invariant：

- Proposal 必须 durable；
- Proposal 创建后 immutable；
- `operations_fingerprint` 绑定 operation 有序列表及其有效 payload；
- AuthorizationResult / PendingHumanGate / AuthorizedDelta 必须引用同一 Proposal；
- 用户若修改审批内容，应创建新 Proposal，而不是篡改旧 Proposal。

---

# 12. SourceRef：我观察了什么

SourceRef 是 Hub Evidence / Source Store 中的可追溯 source observation。

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

一旦被 provenance 引用，`uri/version observation`、`observed_at`、`content_fingerprint`、`supersedes_source_ref_id` 不得原地覆盖。

新观察创建新 SourceRef：

```text
S-004 (commit A)
   ↓ superseded by
S-011 (commit B)
```

`supersedes_source_ref_id` 在新 observation 创建时确定。

---

# 13. EvidenceRef：Source observation 中的证据语义

EvidenceRef 不反向依赖一个已经存在的 State Claim；正式引用方向由 State Entity / Proposal provenance 指向 EvidenceRef。

这样可以避免：

```text
EvidenceRef 必须先创建
但它想引用的 Decision 还没 commit
```

形成循环依赖。

建议字段：

```yaml
evidence_ref:
  id:
  source_ref_id:

  claim_kind:
    # fact | preference | constraint | decision | inference

  stance:
    # supports | contradicts | contextualizes

  authority_role:
    # canonical | supporting | reference

  claim_excerpt_fingerprint:
  created_at:
```

引用方向：

```text
State Entity / Proposal provenance
        ↓
EvidenceRef
        ↓
SourceRef
```

同一个 SourceRef 可以产生多个 EvidenceRef。

---

# 14. Authority Model：四类问题不能混在一起

## 14.1 Source / Epistemic Authority

回答 factual claim 应相信什么证据。不能有全局 `user > document > agent` 排序，必须结合 Claim 类型和 EvidenceRef。

## 14.2 Decision Authority

回答谁有资格把候选结论升级成 Committed Decision。

## 14.3 Action Authority

回答谁可以执行 Continue / Refresh / Rollover / external write 等动作。

## 14.4 Scope Authority

回答谁可以改变 Run 正在解决的问题。

---

# 15. Claim Type 决定 Authority Resolver 的规则

### Normative Claim

用户 goal、preference、scope、product decision、用户明确 constraint。通常以用户/被授权 owner 为高 authority。

### Factual Claim

当前 Git 实现、Production 当前部署、API 当前定义等。应依据 canonical/current evidence，不能套用 `user assertion > source`。

### Inference

Agent 推导默认保持 inference/hypothesis，除非通过 Promotion Policy 进入 Committed State。

---

# 16. Promotion / Authorization Model

```text
Durable Immutable Proposal
   ↓
Authority / Promotion Policy
   ↓
Durable AuthorizationResult
   ├─ authorized
   ├─ requires_human
   └─ denied
```

若 authorized，Policy 产生 Authorized Delta；若 requires_human，Policy 创建 durable PendingHumanGate。

> **LLM proposes; Policy authorizes; Reducer applies; NOOS records.**

---

# 17. AuthorizationResult 与 PendingHumanGate

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

Human approval 后，Policy 基于原 immutable Proposal 重新评估并创建新的 AuthorizationResult。

---

# 18. Review Snapshot / Review Issue

Review Snapshot 绑定 `review_snapshot_id` / `base_state_version`。

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

Stale review 不静默 merge。

---

# 19. State Transition / Audit Records

State Delta Contract 会产生 AuthorizationResult、AuthorizedDelta、ApplyResult、Audit Record。

Proposal、AuthorizationResult、PendingHumanGate 必须 durable；AuthorizedDelta / ApplyResult / Audit Record 至少保存在可追溯的 transition/audit store 中。

---

# 20. Execution Journal 的边界

```text
State Store = 现在是什么
Execution Journal = 执行过程中发生了什么
```

Execution Journal 未来分别处理 dispatch/provider side effect、observation/result capture、reconciliation、state proposal/application。

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

---

# 22. 当前 v0 Invariants

1. Run durable、NOOS-owned。
2. Logical Thread durable，同一时刻至多一个 current Provider Conversation。
3. Provider Conversation replaceable、provider-persisted、provenance-bearing。
4. Browser Session / Adapter Attachment disposable。
5. Committed State 与 Canonical Source 是不同概念。
6. SourceRef 与 EvidenceRef 分离：Source ≠ Claim。
7. Source observation identity immutable。
8. Run State 只 attach SourceRef/EvidenceRef IDs，不由 Reducer改写 observation identity。
9. State Entity / Proposal provenance 单向引用 EvidenceRef；EvidenceRef 不要求反向绑定已存在 State Claim。
10. Normative Claim 与 Factual Claim 使用不同 Authority 规则。
11. Proposal durable 且 immutable；operation 内容变化必须新建 Proposal。
12. LLM 不能直接写 Committed State。
13. `requires_human` 必须形成 durable AuthorizationResult / PendingHumanGate。
14. Reducer 不负责 Authority、事实判断、Human Gate 或 external side effect。
15. Stale Review 必须显式识别，不能静默 merge。

---

# 23. 与 State Delta Contract 的接口

```text
SourceRef
EvidenceRef
Committed State
Working State
Durable Immutable Proposal
AuthorizationResult
PendingHumanGate
Authorized Delta
ApplyResult
```

```text
Proposal = 请求一个最终 State Transaction
Operation = 期望发生的 mutation
Policy = 决定这个事务是否有权发生
Reducer = 在授权后确定性地应用 mutation
```

Authorized Delta 必须绑定原 Proposal 的 `operations_fingerprint`；Policy 不允许在授权过程中偷偷修改 operation 内容。

---

## 当前状态

本文升为 **Design Baseline v0.1**。

经过 Object Model × State Delta 接口对审后，durable ownership、Source/Evidence、Proposal immutability、Authority、Human Gate ownership 与 State transition 边界已经互相闭合。后续若实现/Eval 暴露问题，再通过显式版本变更调整，而不是继续在进入 Continuation 前反复打磨。
