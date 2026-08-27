# NOOS 文档

> 中文工作名当前首选候选：**怒思**  
> 当前重点：NOOS Harness / Execution Plane  
> 当前基线：Harness Design Baseline v0.2  
> 日期：2026-08-27

本仓库用于沉淀 NOOS 的产品需求、架构决策、Harness Runtime、上下文管理与跨工具工作流设计。

## 当前主文档

### Harness

- [NOOS Harness：把 Chatbot 从长对话变成可持续运行的 AI 工作执行器](docs/harness/overview.md)
- [Runtime Object Model & Authority Model v0](docs/harness/runtime-object-authority-model.md)
- [State Delta + Reducer Contract v0](docs/harness/state-delta-reducer-contract.md)

### Branding

- [NOOS / 怒思｜命名与品牌候选记录](docs/branding/naming.md)

## 当前核心判断

> **NOOS 持有工作的连续性、任务状态和显式工作上下文；Chatbot 负责阶段性推理。Provider Conversation 可以替换，Browser Session 可以销毁，而 Run 始终连续。**

当前把 NOOS 看成两个平面：

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

语义上进一步区分：

```text
Canonical Source
→ 决定外部事实的 epistemic authority

Committed State
→ 表示 Run 内已正式提交的约束、决策与 rejection
```

## 当前底层 Contract 顺序

1. **Runtime Object Model & Authority Model v0** — Design Candidate
2. **State Delta + Reducer Contract v0** — Design Candidate
3. **Continuation State Machine v0** — 下一步
4. **Execution Journal & Recovery Contract v0**

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
