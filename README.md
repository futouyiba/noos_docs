# NOOS 文档

> 中文工作名当前首选候选：**怒思**  
> 当前重点：NOOS Harness / Execution Plane  
> 当前基线：Harness Design Baseline v0.2  
> 日期：2026-08-27

本仓库用于沉淀 NOOS 的产品需求、架构决策、Harness Runtime、上下文管理与跨工具工作流设计。

## 当前主文档

### Harness Runtime

- [NOOS Harness：把 Chatbot 从长对话变成可持续运行的 AI 工作执行器](docs/harness/overview.md) — **Design Baseline v0.2**
- [Runtime Object Model & Authority Model v0](docs/harness/runtime-object-authority-model.md) — **Design Baseline v0.1**
- [State Delta + Reducer Contract v0](docs/harness/state-delta-reducer-contract.md) — **Design Baseline v0.1**
- [Continuation State Machine v0](docs/harness/continuation-state-machine.md) — **Design Candidate v0.1**

### Multi-Conversation Orchestration

- [Multi-Conversation Orchestration Role Model v0](docs/harness/multi-conversation-role-model.md) — **Design Candidate v0**

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
3. **Continuation State Machine v0** — **Candidate v0.1；待接口对审**
4. **Execution Journal & Recovery Contract v0** — Continuation 对审通过后进入

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

## Multi-Conversation Orchestration Candidate

当前候选四种 Logical Thread Role：

```text
Control      总控：现在应该做什么？
Design       设计：局部问题怎么解决？
Review       审核：Candidate 哪里有问题？
Integration  集成：如何吸收审核并形成统一 Candidate？
```

关键边界：

```text
Control Thread
≠ Continuation Controller

4 Role Types
≠ 4 Provider Conversations

所有角色修改 Run State
→ Proposal → Policy → Reducer
```

默认生命周期：

```text
Control      → Run-scoped
Design       → Topic / WorkItem-scoped
Review       → Snapshot-scoped
Integration  → Integration-cycle-scoped
```

当前四角色闭环：

```text
Control
→ Design
→ immutable Design Snapshot
→ orthogonal Review(s)
→ Integration
→ State Proposal(s)
→ Policy / Reducer
→ Applied Run State
→ Control
```

`WorkItem` 当前只作为 Candidate correlation object，尚未进入 Runtime Object Model Baseline。

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
