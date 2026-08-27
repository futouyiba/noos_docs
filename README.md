# NOOS 文档

> 中文工作名当前首选候选：**怒思**  
> 当前重点：NOOS Harness / Execution Plane  
> 当前基线：Harness Design Baseline v0.2  
> 日期：2026-08-27

本仓库用于沉淀 NOOS 的产品需求、架构决策、Harness Runtime、上下文管理与跨工具工作流设计。

## 当前主文档

### Harness

- [NOOS Harness：把 Chatbot 从长对话变成可持续运行的 AI 工作执行器](docs/harness/overview.md) — **Design Baseline v0.2**
- [Runtime Object Model & Authority Model v0](docs/harness/runtime-object-authority-model.md) — **Design Baseline v0.1**
- [State Delta + Reducer Contract v0](docs/harness/state-delta-reducer-contract.md) — **Design Baseline v0.1**
- [Continuation State Machine v0](docs/harness/continuation-state-machine.md) — **Design Candidate v0.1**

### Branding

- [NOOS / 怒思｜命名与品牌候选记录](docs/branding/naming.md)

## 当前核心判断

> **NOOS 持有工作的连续性、任务状态和显式工作上下文；Chatbot 负责阶段性推理。Provider Conversation 可以替换，Browser Session 可以销毁，而 Run 始终连续。**

```text
NOOS
├─ Knowledge / Context Plane
│  └─ Vault / Crystal / Handoff / Artifact / Context Broker
│
└─ Execution Plane
   └─ Harness Runtime
```

Harness 当前最关键的控制原则：

> **LLM proposes; Policy authorizes; Reducer applies; NOOS records.**

当前关键语义：

```text
Canonical Source
→ 外部事实的 epistemic authority

Committed State
→ Run 内正式提交的约束、决策与 rejection

SourceRef / EvidenceRef
→ immutable observation 与 claim-level evidence semantics

Proposal
→ durable + immutable 的最小逻辑状态事务请求

State Apply
→ delta_id 幂等；State + ApplyResult + Transition Record crash-consistent commit

ContinuationDecision
→ durable + immutable + basis-fingerprint-idempotent 的“下一步应该做什么”

Execution Journal（下一层）
→ “这个决定后来实际发生了什么”
```

## 当前底层 Contract 顺序

1. **Runtime Object Model & Authority Model v0** — **Baseline v0.1**
2. **State Delta + Reducer Contract v0** — **Baseline v0.1**
3. **Continuation State Machine v0** — **Candidate v0.1；当前进入三篇接口对审**
4. **Execution Journal & Recovery Contract v0** — 对审通过后进入

当前 Continuation 核心模型：

```text
Control Mode
  RUNNING / PAUSED_USER / PAUSED_HUMAN /
  BLOCKED / COMPLETED

+

Execution Phase
  READY / DISPATCH_PENDING / AWAITING_PROVIDER /
  INGESTING_RESULT / SETTLING_STATE / MAINTENANCE
```

几个重要边界：

```text
Transient user typing
→ runtime dispatch guard，不等于 durable PAUSED_USER

WAIT
→ no_action，不制造 durable Decision

Page Health × Context Health
→ 联合决定 Refresh / Compaction / Rollover

Completion candidate
→ 优先进入 completion governance，而不是先做非必要维护
```

Continuation Candidate 还提出需要与 Baseline 对审的对象：

```text
InterventionRequest
HumanGateResolution
ContinuationState
ContinuationDecision
```

其中 Human Gate 允许两种 origin：

```text
PendingHumanGate
├─ AuthorizationResult     # concrete State Proposal 只差 human authority
└─ InterventionRequest     # 先需要选择/输入，尚无唯一 State mutation
```

`Harness Control Block` 只作为 bootstrap transport，不作为独立架构中心。

## MVP 分阶段

```text
M0 Run Continuity Proof
M1 Controlled Rollover
M2 Autonomous Continuation
M3 Performance Self-Healing
M4 Reviewer Orchestration
```

目标是逐层验证价值来源，而不是一次做完整 Runtime v1。
