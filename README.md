# NOOS 文档

> 中文名当前首选候选：**怒思**  
> 当前重点：NOOS Harness / Execution Plane  
> 文档状态：Product Architecture Candidate / v0.1  
> 日期：2026-08-27

本仓库用于沉淀 NOOS 的产品需求、架构决策、Harness Runtime、上下文管理与跨工具工作流设计。

## 当前主文档

- [NOOS Harness：把 Chatbot 从长对话变成可持续运行的 AI 工作执行器](docs/harness/overview.md)

## 当前核心判断

> **NOOS 持有任务状态和显式工作上下文；Chatbot 负责阶段性推理。Conversation 可以刷新、压缩、替换和分叉，而 Run 始终连续。**

当前实现重点不是扩充功能，而是先正式化三个底层契约：

1. Harness Control Contract v0
2. State Delta Schema + Reducer Rules v0
3. Continuation / Recovery State Machine v0
