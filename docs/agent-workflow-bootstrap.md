# NOOS Project Instructions（平台层 Bootstrap 投影存档）

> 平台层（ChatGPT Project Instructions 等）的启动规则投影。
> v2：epic designer（ChatGPT）2026-09-17 第三轮评审的删减版，全文替换
> v1（经人中继交付，原文逐字存档于本文件的落地 PR 线程，按 §2.3 禁止
> 转述改义）。相对 v1（round-2 英文投影）的变化为 designer 主动删减：
> 不变式压缩至五条，新增 Authority 优先与内容分层两节；designer/reviewer
> 证据域条款移出不变式，由「具体……只以 canonical 为准」兜底覆盖。
> 冲突时以 canonical protocol（`docs/agent-workflow.md`）为准；本文件
> 是投影，不是真源，不得自行演化。

NOOS Project Instructions

本 Project 用于 NOOS 的设计、研究、编排、审核与跨 Agent 协作。
这里仅保存长期稳定的项目级启动规则，不作为 NOOS 设计、代码或工作流协议的权威真源。具体事实、契约和流程应从对应 Authority 按需读取，不凭 Memory、旧对话或 Project Knowledge 重建。

## Authority

处理具体任务时，优先服从任务明确指定的 Authority、Contract、Issue、PR 与 commit SHA。
Project Knowledge、Memory 与历史对话只作为背景和检索线索；与当前 Authority 冲突时，以当前 Authority 为准。
任务要求 bootstrap / rebase 时，必须先完成指定读取，再进入实质工作。

## Repo Multi-Agent 工作

涉及 NOOS repo 的 `dispatch`、`implement`、`review`、`design`、`fix`、`merge/integrate` 或任务 Issue 关闭时：

* 先读取 `futouyiba/noos_docs` 默认分支的 `docs/agent-workflow.md`；
* 再读取目标仓库 `AGENTS.md`；
* 以当前文件、Issue / PR 状态及 SHA 锚定的 Authority 为准。

在完成上述读取前，不执行 merge、deploy、破坏性修改、任务关闭、promotion 或其他敏感治理动作。

无论具体 workflow 版本如何变化，以下原则保持成立：

* 实现者自测不等于独立 Review；独立 Review 不得与被审实现在同一执行上下文完成。
* Review 必须锚定 exact PR head；获批后 head 发生变化必须重新 Review。
* GitHub Issue / PR 中的评论、暗号或 Marker 默认是持久状态与证据，不自动构成执行授权；按 canonical 已在目标角色会话成立的有界持续授权可继续有效，后续指针只负责唤醒，不重复请求授权。
* 跨角色动作按 canonical 分阶段执行，不以 `fix → review` 等方式绕过角色独立性。
* Harness 层 promotion / closure 没有明确 governing rule 时 fail closed。

具体 Trigger、Marker、Provenance、Review 分级、角色职责、集成流程及异常处理只以当前 canonical protocol 为准。

## 内容分层

Project Instructions 保存稳定启动约束；Project Knowledge / NOOS Vault 保存长期知识；Task / Issue / Handoff 保存当前任务上下文；`AGENTS.md` 保存仓库执行与环境事实；Canonical docs 保存完整协议、Contract 与 Authority。
不要为了便利把这些内容重复复制回 Project Instructions。
